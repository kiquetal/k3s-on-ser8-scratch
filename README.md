# K3s on Ser8 - Setup Guide

## 1. Obtaining the Operating System

Before setting up Ser8, you need to obtain the appropriate operating system:

1. Visit the official Omakub download page at [https://omakub.org/download](https://omakub.org/download)

2. Download the latest stable version of Omakub OS

3. Verify the ISO image integrity using the SHA256 checksums provided on the download page:
   ```bash
   sha256sum omakub-latest.iso
   ```

4. Compare the output with the official checksum to ensure the ISO hasn't been corrupted.

## 2. Creating a Bootable USB

To install Omakub OS on Ser8, create a bootable USB drive:

### For Linux Users:
```bash
# Identify your USB drive
lsblk

# Write the ISO to the USB drive (replace /dev/sdX with your USB device)
sudo dd bs=4M if=omakub-latest.iso of=/dev/sdX conv=fsync status=progress
```

#### DD Command Parameters Explained:
- `bs=4M`: Sets the block size to 4 megabytes, optimizing transfer speed
- `if=omakub-latest.iso`: Input file - path to the Omakub ISO
- `of=/dev/sdX`: Output file - destination USB drive (replace X with your device letter)
- `conv=fsync`: Forces synchronized I/O to ensure all data is written before command completion
- `status=progress`: Shows real-time progress of the operation

### For Windows Users:
1. Download and install Rufus from [https://rufus.ie](https://rufus.ie)
2. Insert your USB drive
3. Open Rufus and select your USB drive
4. Click on SELECT and choose the Omakub ISO
5. In the "Image option" dropdown, select "Write in DD Image mode"
6. Click START and wait for the process to complete

### For macOS Users:
```bash
# Identify your USB drive
diskutil list

# Unmount the USB drive (replace N with your disk number)
diskutil unmountDisk /dev/diskN

# Write the ISO to the USB drive
sudo dd if=omakub-latest.iso of=/dev/rdiskN bs=1m
```

#### MacOS DD Parameters Explained:
- `if=omakub-latest.iso`: Input file - path to the Omakub ISO
- `of=/dev/rdiskN`: Output file - raw disk device (faster than /dev/diskN)
- `bs=1m`: Block size of 1 megabyte (macOS uses lowercase 'm')

## 3. Setting Up a User on Ser8

After installing Omakub OS on Ser8, create a dedicated user:

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

## 4. Creating SSH Key-Pair from Remote Machine

To securely connect to Ser8 from your remote machine:

1. On your remote machine (laptop/desktop), generate an SSH key pair:
   ```bash
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

2. Copy your public key to Ser8:
   ```bash
   ssh-copy-id k3sadmin@ser8-hostname
   ```
   Alternatively, copy the content of your `~/.ssh/id_ed25519.pub` file and manually add it to the `~/.ssh/authorized_keys` file on Ser8.

3. Test the connection from your remote machine:
   ```bash
   ssh k3sadmin@ser8-hostname
   ```

4. For additional security, configure SSH on Ser8 to disable password authentication:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   
   Change/add the following lines:
   ```
   PasswordAuthentication no
   PubkeyAuthentication yes
   ```

5. Restart the SSH service:
   ```bash
   sudo systemctl restart sshd
   ```

## 5. K3s Installation

For K3s installation, we'll use Ansible. Please refer to the `ansible/` directory for detailed instructions and playbooks.

## Additional Resources

### Installing Omakub

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

### Troubleshooting

If you encounter any issues during the setup process, check the following:
- System logs: `journalctl -xe`
- K3s logs: `sudo journalctl -u k3s`
- Omakub logs: `sudo journalctl -u omakub`
