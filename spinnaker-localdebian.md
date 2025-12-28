Spinnaker Installation Guide (Local Debian on AWS EC2)Updated: December 2025This guide provides a verified workflow for installing Spinnaker on Ubuntu 22.04, including fixes for the "Apache Default Page," "Error Fetching Applications," and "500 Internal Server Error" during application creation.📋 PrerequisitesInstance: Minimum t3.xlarge (16GB RAM).OS: Ubuntu 22.04 LTS.Security Group: Inbound TCP 9000 and 8084 open to your IP.🚀 Installation Steps1. Install Java 17 & HalyardBash# Install Java 17
sudo apt update && sudo apt install -y openjdk-17-jdk
sudo update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java

# Install Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
2. Environment SetupBashsudo mkdir -p /home/spinnaker/.hal/
sudo touch /home/spinnaker/.hal/config
sudo chmod -R 777 /home/spinnaker/
sudo systemctl restart halyard
sleep 30
3. Configure SpinnakerReplace [IP] with your public EC2 IP (e.g., 100.31.139.157).Bashhal config version edit --version 2025.4.0
hal config storage edit --type redis
hal config deploy edit --type localdebian

# 1. Set External URLs
hal config security ui edit --override-base-url http://[IP]:9000
hal config security api edit --override-base-url http://[IP]:8084

# 2. Fix CORS (Prevents "Error Fetching Applications")
hal config security api edit --cors-access-allowed-re http://[IP]:9000

# 3. Disable Fiat (Prevents 500 Error on Application Creation)
hal config security fiat disable

# 4. Force Listeners to 0.0.0.0
mkdir -p ~/.hal/default/service-settings
echo "host: 0.0.0.0" > ~/.hal/default/service-settings/gate.yml
echo "host: 0.0.0.0" > ~/.hal/default/service-settings/deck.yml

# 5. Apply (Note: This resets Apache, proceed to next step immediately after)
sudo hal deploy apply
🛠 Crucial Networking & UI FixesNote: You must run these commands every time you run hal deploy apply because Halyard overwrites the Apache configuration.4. Manual Apache Patch (The "Anti-Revert" Fix)This forces Apache to point to the correct UI folder and listen on the public IP instead of localhost.Bash# 1. Ensure UI permissions (Verify path with: sudo find /opt -name index.html | grep deck)
sudo chmod +x /opt/deck/
sudo chmod -R 755 /opt/deck/html/

# 2. Force Port 9000 Global Listen
echo "Listen 0.0.0.0:9000" | sudo tee /etc/apache2/ports.conf

# 3. Rewrite Spinnaker Site Config
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

# 4. Enable Site and Restart
sudo a2dissite 000-default
sudo a2ensite spinnaker
sudo systemctl restart apache2
🔗 Verification & Maintenance5. Check Ports (Wait 3 Minutes)Gate (8084) is a Java application and takes several minutes to start. Use this to watch them:Bashwatch -n 2 "netstat -tuln | grep -E '8084|9000'"
Correct Result: 0.0.0.0:9000 and :::8084 are both in LISTEN mode.6. Browser AccessAlways use Incognito Mode to bypass Apache and Spinnaker caching.API Health: http://[IP]:8084/health (Should return {"status":"UP"})UI: http://[IP]:9000⚠️ TroubleshootingIssueSolutionApache Default PageYou ran hal deploy apply and it overwrote Step 4. Re-run Step 4 commands.500 Error on Create AppEnsure hal config security fiat disable was run and applied.Error Fetching AppsCheck browser Console (F12). If you see CORS errors, re-verify Step 3.2.403 ForbiddenRun sudo chmod -R 755 /opt/deck/html/.
