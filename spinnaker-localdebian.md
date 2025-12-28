Spinnaker Installation Guide (Local Debian on AWS EC2)
This guide provides a verified, step-by-step workflow to install Spinnaker on a fresh Ubuntu EC2 instance. It includes the necessary Java 17 setup and networking patches required for AWS access.

📋 Prerequisites
Instance Type: Minimum t3.xlarge (4 vCPUs, 16GB RAM).

OS: Ubuntu 22.04 LTS (Jammy).

AWS Security Group: Inbound rules for Ports 9000 (UI) and 8084 (API) from 0.0.0.0/0.

🚀 Installation Steps
1. Install Java 17 (Required for Halyard 2025+)
Modern Halyard versions require Java 17. Check and set this before installing Halyard to avoid UnsupportedClassVersionError.

Bash

# Install OpenJDK 17
sudo apt update
sudo apt install -y openjdk-17-jdk

# Ensure Java 17 is the default
sudo update-alternatives --config java
# (Select the number corresponding to Java 17 if prompted)

# Verify
java -version
2. Install Halyard
Bash

# Install curl
sudo apt-get install -y curl

# Download and run the Halyard installer
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh

# Verify installation
hal -v
3. Environment & Permissions Setup
Fix permissions to allow the spinnaker user to write configuration files.

Bash

sudo mkdir -p /home/spinnaker/.hal/
sudo touch /home/spinnaker/.hal/config
sudo chmod -R 777 /home/spinnaker/

# Restart Halyard and wait for the Java daemon to warm up
sudo systemctl restart halyard
echo "Waiting 45 seconds for Halyard..."
sleep 45
4. Configure Spinnaker
Note: Replace [YOUR_PUBLIC_IP] with your EC2 instance's Public IP.

Bash

# 1. Set Version
hal config version edit --version 2025.4.0

# 2. Set Storage and Type
hal config storage edit --type redis
hal config deploy edit --type localdebian

# 3. Configure External Access URLs
hal config security ui edit --override-base-url http://[YOUR_PUBLIC_IP]:9000
hal config security api edit --override-base-url http://[YOUR_PUBLIC_IP]:8084

# 4. Force Gate (API) to listen on all interfaces
mkdir -p ~/.hal/default/service-settings
echo "host: 0.0.0.0" > ~/.hal/default/service-settings/gate.yml

# 5. Enable Kubernetes Provider
hal config provider kubernetes enable
5. Deploy Spinnaker
Bash

sudo hal deploy apply
6. Networking & Apache Patch (Crucial for UI Access)
Halyard defaults Apache to 127.0.0.1. Use these commands to force port 9000 to be public and enable the Spinnaker site.

Bash

# Force Apache to listen globally
echo "Listen 0.0.0.0:9000" | sudo tee /etc/apache2/ports.conf

# Enable the Spinnaker site and disable the default "It Works" page
sudo a2dissite 000-default
sudo a2ensite spinnaker

# Update the Spinnaker VirtualHost
sudo sed -i 's/127.0.0.1:9000/*:9000/g' /etc/apache2/sites-available/spinnaker.conf

# Final Restart
sudo systemctl daemon-reload
sudo systemctl restart apache2
sudo systemctl restart spinnaker
🛠 Troubleshooting
Gate (8084) is not listening
If netstat -tuln | grep 8084 shows nothing, the service is still booting (takes 2-3 mins). If it shows 127.0.0.1, ensure your gate.yml override is in the correct path:

Bash

sudo mkdir -p /home/spinnaker/.hal/default/service-settings
echo "host: 0.0.0.0" | sudo tee /home/spinnaker/.hal/default/service-settings/gate.yml
sudo hal deploy apply
Apache "Default Page" instead of Spinnaker
If you see the "Apache2 Ubuntu Default Page" on port 9000, run:

Bash

sudo a2ensite spinnaker
sudo systemctl restart apache2
🔗 Verification
UI: http://[YOUR_PUBLIC_IP]:9000

API Health: http://[YOUR_PUBLIC_IP]:8084/health
