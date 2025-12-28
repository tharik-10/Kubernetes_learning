Spinnaker Installation Guide (Local Debian on AWS EC2)Updated: December 2025This guide provides a verified workflow to install Spinnaker, including the specific networking and file-path patches required for Ubuntu 22.04.📋 PrerequisitesInstance: Minimum t3.xlarge (16GB RAM).OS: Ubuntu 22.04 LTS.Security Group: Inbound TCP 9000 and 8084 open to your IP.🚀 Installation Steps1. Install Java 17 & HalyardBash# 1. Install Java 17
sudo apt update && sudo apt install -y openjdk-17-jdk
sudo update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java

# 2. Install Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
2. Environment SetupBashsudo mkdir -p /home/spinnaker/.hal/
sudo touch /home/spinnaker/.hal/config
sudo chmod -R 777 /home/spinnaker/
sudo systemctl restart halyard
sleep 30
3. Configure SpinnakerReplace [IP] with your public EC2 IP.Bashhal config version edit --version 2025.4.0
hal config storage edit --type redis
hal config deploy edit --type localdebian

# Set External URLs
hal config security ui edit --override-base-url http://[IP]:9000
hal config security api edit --override-base-url http://[IP]:8084

# Force Listeners to 0.0.0.0 (Override Halyard defaults)
mkdir -p ~/.hal/default/service-settings
echo "host: 0.0.0.0" > ~/.hal/default/service-settings/gate.yml
echo "host: 0.0.0.0" > ~/.hal/default/service-settings/deck.yml

sudo hal deploy apply
🛠 Crucial Networking & UI FixesHalyard often misconfigures Apache paths and IP bindings. These steps are mandatory.4. Fix Apache Permissions & PathsIdentify where your UI files are (usually /opt/deck/html/) and grant access:Bash# Grant traversal and read permissions
sudo chmod +x /opt/deck/
sudo chmod -R 755 /opt/deck/html/
5. Manual Apache ConfigurationHalyard's generated config often points to the wrong folder. Overwrite it with this "Hard Fix":Bash# 1. Force Port 9000
echo "Listen 9000" | sudo tee /etc/apache2/ports.conf

# 2. Re-write the Spinnaker Site Config
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

# 3. Disable Default and Enable Spinnaker
sudo a2dissite 000-default
sudo a2ensite spinnaker
sudo systemctl restart apache2
🔗 VerificationCheck Ports: Run netstat -tuln | grep -E '9000|8084'. Both should show 0.0.0.0 or :::.API Health: Visit http://[IP]:8084/health. Should return {"status":"UP"}.UI Access: Visit http://[IP]:9000 in Incognito Mode (to bypass Apache caching).⚠️ TroubleshootingIssueSolution403 ForbiddenRun sudo chmod -R 755 /opt/deck/html/ and check Apache <Directory> settings.Apache Default PageRun sudo a2dissite 000-default and restart Apache.Gate (8084) missingGate takes 3+ minutes to start. Check logs: sudo journalctl -u gate -f.UI says "Offline"Re-run hal config security api edit with the correct IP and hal deploy apply.
