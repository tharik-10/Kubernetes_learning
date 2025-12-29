🛠 Phase 1: AWS Infrastructure (Pre-Setup)Before touching the terminal, ensure your AWS Console looks like this:ComponentSettingValueTarget Group 1Name: spin-deck-tgPort: 9000 / Protocol: HTTP / Health: /Target Group 2Name: spin-gate-tgPort: 8084 / Protocol: HTTP / Health: /healthALB Listener 1Port 9000Forward to spin-deck-tgALB Listener 2Port 8084Forward to spin-gate-tgEC2 Sec GroupInboundTCP 9000, 8084, 8080 from 0.0.0.0/0 (or ALB SG)🚀 Phase 2: System Optimization & InstallationRun these as the ubuntu user.1. Create a Swap File (Prevents Crashes)Even with 16GB RAM, Spinnaker spikes. This 4GB "safety net" prevents the Killed error.Bashsudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
2. Install Halyard & Java 17Bashsudo apt update && sudo apt install -y openjdk-17-jdk net-tools apache2
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh -y --user ubuntu
sleep 20
📦 Phase 3: Persistent Storage (S3)Using Redis for storage is dangerous. Use S3 so your pipelines survive reboots.Generate an AWS Access Key/Secret for a user with S3 Full Access.Bashexport MY_ACCESS_KEY="your_access_key"
export MY_SECRET_KEY="your_secret_key"
export MY_BUCKET="spinnaker-storage-$(date +%s)" # Unique bucket name

echo $MY_SECRET_KEY | hal config storage s3 edit \
    --access-key-id $MY_ACCESS_KEY \
    --secret-access-key \
    --region us-east-1 \
    --bucket $MY_BUCKET

hal config storage edit --type s3
🔒 Phase 4: Networking & URL LockdownReplace YOUR_ALB_DNS with your AWS Load Balancer DNS name.Bashexport ALB_DNS="spinnaker-1357806631.us-east-1.elb.amazonaws.com"

hal config version edit --version 2025.4.0
hal config deploy edit --type localdebian

# Set External URLs
hal config security ui edit --override-base-url http://$ALB_DNS:9000
hal config security api edit --override-base-url http://$ALB_DNS:8084
hal config security api edit --cors-access-allowed-origin http://$ALB_DNS:9000
Force 0.0.0.0 Binding (The Fix for Timeouts)Bash# Force Gate (API)
sudo mkdir -p /etc/systemd/system/gate.service.d/
echo -e "[Service]\nEnvironment=\"JAVA_OPTS=-Dserver.address=0.0.0.0\"" | sudo tee /etc/systemd/system/gate.service.d/override.conf

# Force Front50
sudo mkdir -p /etc/systemd/system/front50.service.d/
echo -e "[Service]\nEnvironment=\"JAVA_OPTS=-Dserver.address=0.0.0.0\"" | sudo tee /etc/systemd/system/front50.service.d/override.conf

sudo systemctl daemon-reload
🌐 Phase 5: Apache Web Server (UI)Deck needs Apache to serve files on port 9000.Bash# Configure Apache Ports
sudo sed -i 's/Listen 80/Listen 9000/' /etc/apache2/ports.conf

# Create Site Config
sudo tee /etc/apache2/sites-available/spinnaker.conf <<EOF
<VirtualHost *:9000>
  DocumentRoot /opt/deck/html/
  <Directory /opt/deck/html/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
  </Directory>
</VirtualHost>
EOF

sudo a2dissite 000-default.conf || true
sudo a2ensite spinnaker.conf
sudo systemctl restart apache2
🏁 Phase 6: Deployment & Final CheckBashsudo hal deploy apply

# CRITICAL: Restart services manually to ensure the 0.0.0.0 override takes effect
sudo systemctl restart redis-server front50 gate
🩺 How to verify success:Wait 5-8 minutes. Java takes time.Run sudo netstat -tuln | grep -E '8084|9000'. You must see :::8084 and :::9000.Check Gate Health: curl -I http://localhost:8084/health. It must say 200 OK.Open your browser to http://YOUR_ALB_DNS:9000.Common Issue: If it still spins, open your browser "Developer Tools" (F12) -> Console. If you see "CORS Error", re-run the hal config security api edit --cors-access-allowed-origin command and hal deploy apply one more time.
