# K3s on Ser8 - Setup Guide

## First-time Setup on Ser8

This section provides instructions for setting up K3s on the Ser8 system for the first time.

### Prerequisites

- Access to Ser8 system
- Administrator privileges
- Basic knowledge of Linux commands

### Installation Steps

1. Update the system packages:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. Install required dependencies:
   ```bash
   sudo apt install -y curl openssh-server
   ```

3. Download and install K3s:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```

4. Verify the installation:
   ```bash
   sudo kubectl get nodes
   ```

## Installing Omakub

Omakub can be installed from their official website. Follow these steps to install Omakub:

1. Visit the official Omakub website at [https://omakub.io/downloads](https://omakub.io/downloads)

2. Select the Linux distribution package compatible with Ser8 (typically the Debian/Ubuntu package)

3. Download the package using:
   ```bash
   wget https://omakub.io/downloads/omakub_latest_amd64.deb
   ```

4. Install the package:
   ```bash
   sudo dpkg -i omakub_latest_amd64.deb
   ```

5. If there are dependency issues, resolve them with:
   ```bash
   sudo apt --fix-broken install -y
   ```

6. Verify the installation:
   ```bash
   omakub version
   ```

7. Configure Omakub to work with your K3s installation:
   ```bash
   sudo omakub setup --k3s-kubeconfig /etc/rancher/k3s/k3s.yaml
   ```

## Creating an SSH User for Remote Access

To connect to Ser8 from your laptop, follow these steps to create a dedicated SSH user:

1. Create a new user on Ser8:
   ```bash
   sudo adduser k3sadmin
   ```

2. Add the user to the sudo group for administrative privileges:
   ```bash
   sudo usermod -aG sudo k3sadmin
   ```

3. Switch to the new user:
   ```bash
   su - k3sadmin
   ```

4. Create an SSH directory and set appropriate permissions:
   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   ```

5. Create the authorized_keys file for storing your public key:
   ```bash
   touch ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

6. On your laptop, generate an SSH key pair if you don't already have one:
   ```bash
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

7. Copy your laptop's public key to the server:
   ```bash
   ssh-copy-id k3sadmin@ser8-hostname
   ```
   Alternatively, manually add the public key to `~/.ssh/authorized_keys` on the server.

8. Test the connection from your laptop:
   ```bash
   ssh k3sadmin@ser8-hostname
   ```

9. For additional security, consider configuring SSH to disable password authentication and use only key-based authentication.

## Next Steps

After completing the above setup, you can proceed to:
- Deploy applications on your K3s cluster
- Configure networking and storage
- Set up monitoring and logging

## Troubleshooting

If you encounter any issues during the setup process, check the following:
- System logs: `journalctl -xe`
- K3s logs: `sudo journalctl -u k3s`
- Omakub logs: `sudo journalctl -u omakub`
