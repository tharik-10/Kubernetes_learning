⚓ Spinnaker Production-Ready Guide (ALB + EC2)Updated: December 2025🛠 Phase 1: AWS Infrastructure SetupConfigure your AWS Console exactly as follows before beginning installation.ComponentSettingValueTarget Group 1Namespin-deck-tgPort / Protocol9000 / HTTPHealth Check Path/Target Group 2Namespin-gate-tgPort / Protocol8084 / HTTPHealth Check Path/health (Must be explicit)ALB ListenersHTTP:9000Forward to spin-deck-tgHTTP:8084Forward to spin-gate-tgSecurity GroupInboundTCP 9000 & 8084 from ALB SG🚀 Phase 2: Installation & Core ConfigurationConnect to your EC2 instance and execute these steps.1. System PreparationBashsudo apt update && sudo apt install -y openjdk-17-jdk net-tools
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
# Wait for the Halyard daemon to initialize
sleep 20
2. External URL & CORS ConfigurationReplace your-alb-dns-name.amazonaws.com with your actual ALB DNS.Bashexport ALB_URL="your-alb-dns-name.amazonaws.com"

hal config version edit --version 2025.4.0
hal config storage edit --type redis
hal config deploy edit --type localdebian

# Configure External Access
hal config security ui edit --override-base-url http://$ALB_URL:9000
hal config security api edit --override-base-url http://$ALB_URL:8084
hal config security api edit --cors-access-allowed-origin http://$ALB_URL:9000
🔒 Phase 3: The Binding Lockdown (Permanent Fix)This phase ensures Spinnaker services listen on the Private IP (0.0.0.0) so the ALB can reach them.1. Force Gate (API) to 0.0.0.0We use a Systemd override to ensure Java binds correctly, regardless of config files.Bashsudo mkdir -p /etc/systemd/system/gate.service.d/
echo -e "[Service]\nEnvironment=\"JAVA_OPTS=-Dserver.address=0.0.0.0\"" | sudo tee /etc/systemd/system/gate.service.d/override.conf

sudo systemctl daemon-reload
2. Force Deck (UI) via ApacheDeck is served by Apache. We must tell Apache to listen on all interfaces for port 9000.Bash# Update Apache ports
echo "Listen 9000" | sudo tee /etc/apache2/ports.conf

# Create the Spinnaker Site Config
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

# Activate site and restart
sudo a2dissite 000-default.conf || true
sudo a2ensite spinnaker.conf
sudo systemctl restart apache2
3. Apply and DeployBashsudo hal deploy apply
sudo systemctl restart gate
🔗 Phase 4: Final VerificationExecute these checks to confirm the setup is live.1. Check Network BindingsRun: sudo netstat -tuln | grep -E '8084|9000'Correct Output:0.0.0.0:9000 or :::9000 — OK0.0.0.0:8084 or :::8084 — OK127.0.0.1:XXXX — FAIL (Service is still private)2. Check Gate HealthVerify via browser or curl: http://[YOUR_ALB_URL]:8084/healthSuccess: {"status": "UP"}⚠️ Troubleshooting "Infinite Spinning Logo"If the UI loads but stays on a spinning logo:CORS: Re-run the hal config security api edit --cors-access-allowed-origin command from Phase 2 and hal deploy apply.Browser Cache: Browsers aggressively cache Spinnaker's older "localhost" settings. Always test in an Incognito/Private window.
