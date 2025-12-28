Spinnaker Installation Guide (Local Debian on AWS EC2)
This guide provides a verified, step-by-step workflow to install Spinnaker on a fresh Ubuntu EC2 instance using the Halyard CLI. It includes specific fixes for AWS networking and common permission issues.

📋 Prerequisites
Instance Type: Minimum t3.xlarge (4 vCPUs, 16GB RAM).

OS: Ubuntu 20.04 or 22.04 LTS.

AWS Security Group: Inbound rules for Ports 9000 (UI) and 8084 (API) from 0.0.0.0/0.

🚀 Installation Steps
1. Install Halyard
Halyard is the administration tool used to configure and deploy Spinnaker.

Bash

# Update system and install curl
sudo apt-get update && sudo apt-get install -y curl

# Download and run the Halyard installer
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh

# Verify installation (you may need to restart your shell)
hal -v
2. Environment & Permissions Setup
Halyard requires a specific directory structure. We force these permissions to prevent "Daemon Connectivity" and "Permission Denied" errors.

Bash

# Create the config directory Halyard expects
sudo mkdir -p /home/spinnaker/.hal/
sudo touch /home/spinnaker/.hal/config
sudo chmod -R 777 /home/spinnaker/

# Restart the Halyard daemon and wait for Java to initialize
sudo systemctl restart halyard
echo "Waiting 45 seconds for the Halyard Daemon to boot..."
sleep 45
3. Configure Spinnaker
Run these commands one by one to configure the deployment.

Note: Replace [YOUR_PUBLIC_IP] with your EC2 instance's actual Public IP address.

Bash

# Set the Spinnaker version
hal config version edit --version 2025.4.0

# Set storage and deployment type
hal config storage edit --type redis
hal config deploy edit --type localdebian

# Enable Kubernetes provider
hal config provider kubernetes enable

# Bind services to 0.0.0.0 for external access
hal config edit --host 0.0.0.0

# Configure UI and API Endpoints for your Public IP
hal config security ui edit --override-base-url http://[YOUR_PUBLIC_IP]:9000
hal config security api edit --override-base-url http://[YOUR_PUBLIC_IP]:8084
4. Deploy Spinnaker
This command triggers the actual download and installation of the microservices.

Bash

sudo hal deploy apply
5. Networking Patch (AWS Access)
Halyard's default Apache configuration often locks to 127.0.0.1. Use these commands to force it to listen on the public interface.

Bash

# Update Apache ports configuration
sudo sed -i 's/Listen 127.0.0.1:9000/Listen 0.0.0.0:9000/g' /etc/apache2/ports.conf

# Update Spinnaker site configuration
sudo sed -i 's/<VirtualHost 127.0.0.1:9000>/<VirtualHost *:9000>/g' /etc/apache2/sites-available/spinnaker.conf

# Restart services to apply changes
sudo systemctl daemon-reload
sudo systemctl restart apache2
sudo systemctl restart spinnaker
🛠 Troubleshooting
Infinite Loading UI / 500 Errors
If you can access the UI but cannot create an application, disable authorization checks (Fiat):

Bash

hal config security authz disable
sudo hal deploy apply
Halyard Daemon Connection Issues
If you see Failed to connect to localhost/127.0.0.1:8064, ensure the service is running and has enough RAM:

Bash

sudo systemctl restart halyard
sudo journalctl -u halyard -f # To watch for errors
🔗 Verification
Access your Spinnaker instance at:

UI: http://[YOUR_PUBLIC_IP]:9000

API: http://[YOUR_PUBLIC_IP]:8084/health
