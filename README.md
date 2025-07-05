# K3s on Ser8 with Omakub OS - Setup Guide

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

## 3. Installing Omakub OS on Ser8

1. Insert the bootable USB drive into your Ser8 device
2. Power on Ser8 and enter the BIOS/UEFI (usually by pressing F2, DEL, or ESC during startup)
3. Set the USB drive as the first boot device
4. Save and exit the BIOS/UEFI
5. When the Omakub installer appears, follow the on-screen instructions
6. Select your preferred language, time zone, and keyboard layout
7. When prompted for installation type, choose "Install Omakub OS"
8. At the partitioning step, select "Manual partitioning" for optimal Kubernetes and Docker performance

### 3.1 Recommended Partition Scheme for K8s and Docker

When manually partitioning the disk, follow these recommendations:

| Partition | Mount Point | File System | Size | Purpose |
|-----------|-------------|------------|------|---------|
| /dev/sda1 | /boot/efi  | FAT32      | 512 MB | EFI System Partition |
| /dev/sda2 | /boot      | ext4       | 1 GB   | Boot files |
| /dev/sda3 | /          | ext4       | 40 GB  | Root filesystem |
| /dev/sda4 | /var       | ext4       | 30 GB  | Variable data (logs, cache) |
| /dev/sda5 | /var/lib/docker | ext4  | 60-100 GB | Docker storage |
| /dev/sda6 | /var/lib/rancher | ext4 | 30-50 GB | K3s data |
| /dev/sda7 | swap       | swap       | 8 GB   | Swap space |

#### Partition Rationale

- **Separate /var partition**: Prevents logs and temporary files from filling the root partition
- **Dedicated Docker storage**: Isolates container images and volumes for better performance
- **Dedicated K3s storage**: Isolates Kubernetes data for better performance
- **Appropriate sizing**: Ensures adequate space for container images and Kubernetes components

#### Performance Considerations

- For SSDs, add the `noatime` mount option to reduce unnecessary write operations
- If using an NVMe drive, consider using the `ext4` file system with `discard=async` for better performance
- For mechanical HDDs, consider setting up an SSD cache for `/var/lib/docker` to improve container startup times

9. Complete the installation process and reboot when prompted
10. Remove the USB drive during reboot

### 3.2 Step-by-Step Omakub Partitioning Guide

Omakub OS provides a specialized partitioning tool that simplifies disk setup for Kubernetes and container workloads. Here's how to use it:

1. **Launch Omakub Partitioning Tool**:
   - During installation, when you reach the partitioning step, select "Manual Partitioning"
   - Click on "Advanced Mode" to access the Omakub Disk Manager

2. **Using the Omakub Disk Manager**:

   a. **Select Target Disk**:
   ```bash
   # The tool will display available disks:
   omakub-disk-manager --list-disks

   # Select your target installation disk
   omakub-disk-manager --select /dev/sda
   ```

   b. **Use the K8s Optimized Template**:
   ```bash
   # For Kubernetes and Docker optimized layout
   omakub-disk-manager --apply-template k8s-server
   ```

   c. **Customize Partition Sizes** (if needed):
   ```bash
   # Syntax: omakub-disk-manager --resize <mount-point> <size-in-GB>
   omakub-disk-manager --resize /var/lib/docker 80
   omakub-disk-manager --resize /var/lib/rancher 40
   ```

   d. **Apply SSD Optimizations** (if using SSD/NVMe):
   ```bash
   omakub-disk-manager --optimize ssd
   ```

3. **Review Partition Layout**:
   ```bash
   # Verify partition configuration before applying
   omakub-disk-manager --show-plan
   ```

4. **Apply the Partition Configuration**:
   ```bash
   # Execute the partitioning plan
   omakub-disk-manager --apply
   ```

#### Omakub GUI Partitioning

If you prefer using the graphical interface:

1. In the Omakub installer, select "Manual Partitioning"
2. Click on "Omakub Disk Assistant"
3. From the dropdown menu, select "Kubernetes Server Profile"
4. The assistant will automatically create the recommended partition layout
5. You can adjust partition sizes using the sliders
6. Check the "Optimize for SSD" option if applicable
7. Click "Apply" to confirm your partition layout

#### Omakub Partition Verification

After applying the partition layout, you can verify it using:

```bash
# From the installation environment
omakub-verify-layout --profile=k8s

# This will check if your partitioning follows best practices for Kubernetes
```

The verification tool will provide feedback and optimization suggestions based on your specific hardware.

### 3.3 Partition Encryption Options

For enhanced security, Omakub supports disk encryption:

1. **Full Disk Encryption**:
   - Select "Enable Encryption" in the partition manager
   - Choose a strong passphrase when prompted
   - Note: The `/boot` partition remains unencrypted

2. **Selective Partition Encryption**:
   - In advanced mode, you can selectively encrypt partitions:
   ```bash
   omakub-disk-manager --encrypt /var/lib/docker
   omakub-disk-manager --encrypt /var/lib/rancher
   ```

3. **Performance Considerations**:
   - Encryption adds a small overhead, especially with mechanical HDDs
   - Modern CPUs with AES-NI support minimize this impact on SSDs
   - For optimal performance, consider encrypting only sensitive partitions

Continue with the installation process after completing the partitioning.

## 4. Setting Up a User on Ser8

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

## 5. Creating SSH Key-Pair from Remote Machine

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

## 6. K3s Installation

For K3s installation on Omakub OS, we'll use Ansible. Please refer to the `ansible/` directory for detailed instructions and playbooks.

## Additional Resources

### Managing Omakub OS

Omakub OS includes several tools to help manage your system:

1. System updates:
   ```bash
   sudo omakub-update
   ```

2. Package management:
   ```bash
   sudo omakub-pkg install <package-name>
   sudo omakub-pkg remove <package-name>
   sudo omakub-pkg list
   ```

3. System monitoring:
   ```bash
   omakub-monitor
   ```

4. Configure Omakub to work with your K3s installation:
   ```bash
   sudo omakub setup --k3s-kubeconfig /etc/rancher/k3s/k3s.yaml
   ```

### Troubleshooting

If you encounter any issues during the setup process, check the following:
- System logs: `journalctl -xe`
- K3s logs: `sudo journalctl -u k3s`
- Omakub logs: `sudo journalctl -u omakub`
- Omakub system diagnostics: `sudo omakub-diag`

# K3s on Ser8 with Omakub OS - Setup Guide

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

## 3. Installing Omakub OS on Ser8

1. Insert the bootable USB drive into your Ser8 device
2. Power on Ser8 and enter the BIOS/UEFI (usually by pressing F2, DEL, or ESC during startup)
3. Set the USB drive as the first boot device
4. Save and exit the BIOS/UEFI
5. When the Omakub installer appears, follow the on-screen instructions
6. Select your preferred language, time zone, and keyboard layout
7. When prompted for installation type, choose "Install Omakub OS"
8. At the partitioning step, select "Manual partitioning" for optimal Kubernetes and Docker performance

### 3.1 Recommended Partition Scheme for K8s and Docker

When manually partitioning the disk, follow these recommendations:

| Partition | Mount Point | File System | Size | Purpose |
|-----------|-------------|------------|------|---------|
| /dev/sda1 | /boot/efi  | FAT32      | 512 MB | EFI System Partition |
| /dev/sda2 | /boot      | ext4       | 1 GB   | Boot files |
| /dev/sda3 | /          | ext4       | 40 GB  | Root filesystem |
| /dev/sda4 | /var       | ext4       | 30 GB  | Variable data (logs, cache) |
| /dev/sda5 | /var/lib/docker | ext4  | 60-100 GB | Docker storage |
| /dev/sda6 | /var/lib/rancher | ext4 | 30-50 GB | K3s data |
| /dev/sda7 | swap       | swap       | 8 GB   | Swap space |

#### Partition Rationale

- **Separate /var partition**: Prevents logs and temporary files from filling the root partition
- **Dedicated Docker storage**: Isolates container images and volumes for better performance
- **Dedicated K3s storage**: Isolates Kubernetes data for better performance
- **Appropriate sizing**: Ensures adequate space for container images and Kubernetes components

#### Performance Considerations

- For SSDs, add the `noatime` mount option to reduce unnecessary write operations
- If using an NVMe drive, consider using the `ext4` file system with `discard=async` for better performance
- For mechanical HDDs, consider setting up an SSD cache for `/var/lib/docker` to improve container startup times

9. Complete the installation process and reboot when prompted
10. Remove the USB drive during reboot

### 3.2 Step-by-Step Omakub Partitioning Guide

Omakub OS provides a specialized partitioning tool that simplifies disk setup for Kubernetes and container workloads. Here's how to use it:

1. **Launch Omakub Partitioning Tool**:
   - During installation, when you reach the partitioning step, select "Manual Partitioning"
   - Click on "Advanced Mode" to access the Omakub Disk Manager

2. **Using the Omakub Disk Manager**:

   a. **Select Target Disk**:
   ```bash
   # The tool will display available disks:
   omakub-disk-manager --list-disks

   # Select your target installation disk
   omakub-disk-manager --select /dev/sda
   ```

   b. **Use the K8s Optimized Template**:
   ```bash
   # For Kubernetes and Docker optimized layout
   omakub-disk-manager --apply-template k8s-server
   ```

   c. **Customize Partition Sizes** (if needed):
   ```bash
   # Syntax: omakub-disk-manager --resize <mount-point> <size-in-GB>
   omakub-disk-manager --resize /var/lib/docker 80
   omakub-disk-manager --resize /var/lib/rancher 40
   ```

   d. **Apply SSD Optimizations** (if using SSD/NVMe):
   ```bash
   omakub-disk-manager --optimize ssd
   ```

3. **Review Partition Layout**:
   ```bash
   # Verify partition configuration before applying
   omakub-disk-manager --show-plan
   ```

4. **Apply the Partition Configuration**:
   ```bash
   # Execute the partitioning plan
   omakub-disk-manager --apply
   ```

#### Omakub GUI Partitioning

If you prefer using the graphical interface:

1. In the Omakub installer, select "Manual Partitioning"
2. Click on "Omakub Disk Assistant"
3. From the dropdown menu, select "Kubernetes Server Profile"
4. The assistant will automatically create the recommended partition layout
5. You can adjust partition sizes using the sliders
6. Check the "Optimize for SSD" option if applicable
7. Click "Apply" to confirm your partition layout

#### Omakub Partition Verification

After applying the partition layout, you can verify it using:

```bash
# From the installation environment
omakub-verify-layout --profile=k8s

# This will check if your partitioning follows best practices for Kubernetes
```

The verification tool will provide feedback and optimization suggestions based on your specific hardware.

### 3.3 Partition Encryption Options

For enhanced security, Omakub supports disk encryption:

1. **Full Disk Encryption**:
   - Select "Enable Encryption" in the partition manager
   - Choose a strong passphrase when prompted
   - Note: The `/boot` partition remains unencrypted

2. **Selective Partition Encryption**:
   - In advanced mode, you can selectively encrypt partitions:
   ```bash
   omakub-disk-manager --encrypt /var/lib/docker
   omakub-disk-manager --encrypt /var/lib/rancher
   ```

3. **Performance Considerations**:
   - Encryption adds a small overhead, especially with mechanical HDDs
   - Modern CPUs with AES-NI support minimize this impact on SSDs
   - For optimal performance, consider encrypting only sensitive partitions

Continue with the installation process after completing the partitioning.

### 3.4 Modern Partitioning for Beelink SER8

The Beelink SER8 typically comes with NVMe storage, which benefits from specific partitioning approaches. Here's how to create the optimized partition layout using modern tools:

#### Using parted and systemd-mount (Recommended for NVMe)

1. **Identify your NVMe drive**:
   ```bash
   lsblk -o NAME,SIZE,TRAN,MODEL
   ```
   
   Typically, NVMe drives appear as `/dev/nvme0n1`

2. **Create a GPT partition table**:
   ```bash
   sudo parted /dev/nvme0n1 mklabel gpt
   ```

3. **Create the partitions using parted**:
   ```bash
   # Create EFI partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "EFI" fat32 0% 512MiB
   sudo parted /dev/nvme0n1 set 1 esp on
   
   # Create boot partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "boot" ext4 512MiB 1.5GiB
   
   # Create root partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "root" ext4 1.5GiB 41.5GiB
   
   # Create var partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "var" ext4 41.5GiB 71.5GiB
   
   # Create docker storage partition (adjust size based on your NVMe capacity)
   sudo parted -a optimal /dev/nvme0n1 mkpart "docker" ext4 71.5GiB 151.5GiB
   
   # Create k3s data partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "k3s" ext4 151.5GiB 191.5GiB
   
   # Create swap partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "swap" linux-swap 191.5GiB 199.5GiB
   ```

4. **Format the partitions with optimizations for NVMe**:
   ```bash
   # Format EFI partition
   sudo mkfs.fat -F32 /dev/nvme0n1p1
   
   # Format boot with reserved space for system stability
   sudo mkfs.ext4 -L boot /dev/nvme0n1p2
   
   # Format root with optimizations
   sudo mkfs.ext4 -L root -O fast_commit,extent,dir_index /dev/nvme0n1p3
   
   # Format var with journal size optimization
   sudo mkfs.ext4 -L var -O fast_commit,extent,dir_index -J size=64 /dev/nvme0n1p4
   
   # Format docker with optimizations for containers
   sudo mkfs.ext4 -L docker -i 8192 -I 256 -O fast_commit,extent,dir_index,64bit /dev/nvme0n1p5
   
   # Format k3s with optimizations for Kubernetes
   sudo mkfs.ext4 -L k3s -i 8192 -O fast_commit,extent,dir_index /dev/nvme0n1p6
   
   # Create and enable swap
   sudo mkswap -L swap /dev/nvme0n1p7
   sudo swapon /dev/nvme0n1p7
   ```

5. **Mount partitions with NVMe optimizations**:
   ```bash
   # Mount root
   sudo mount -o defaults,noatime,discard=async /dev/nvme0n1p3 /mnt
   
   # Create mount points
   sudo mkdir -p /mnt/{boot/efi,var,var/lib/docker,var/lib/rancher}
   
   # Mount other partitions with optimizations
   sudo mount -o defaults /dev/nvme0n1p1 /mnt/boot/efi
   sudo mount -o defaults,noatime /dev/nvme0n1p2 /mnt/boot
   sudo mount -o defaults,noatime,discard=async /dev/nvme0n1p4 /mnt/var
   sudo mount -o defaults,noatime,discard=async,nobarrier /dev/nvme0n1p5 /mnt/var/lib/docker
   sudo mount -o defaults,noatime,discard=async /dev/nvme0n1p6 /mnt/var/lib/rancher
   ```

6. **Generate optimized fstab**:
   ```bash
   sudo mkdir -p /mnt/etc
   sudo bash -c 'echo "# /etc/fstab: system mounts" > /mnt/etc/fstab'
   
   # Add entries with optimal mount options
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1) /boot/efi vfat defaults 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p2) /boot ext4 defaults,noatime 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3) / ext4 defaults,noatime,discard=async 0 1" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p4) /var ext4 defaults,noatime,discard=async 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p5) /var/lib/docker ext4 defaults,noatime,discard=async,nobarrier 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p6) /var/lib/rancher ext4 defaults,noatime,discard=async 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p7) none swap sw 0 0" | sudo tee -a /mnt/etc/fstab
   ```

#### SER8-Specific Optimization

The Beelink SER8's NVMe drive benefits from additional optimizations:

1. **I/O Scheduler Configuration**:
   ```bash
   # After installation, set the scheduler to none for NVMe (best performance)
   echo 'ACTION=="add|change", KERNEL=="nvme0n1", ATTR{queue/scheduler}="none"' | sudo tee /etc/udev/rules.d/60-schedulers.rules
   ```

2. **Memory and Swap Optimization for Kubernetes**:
   ```bash
   # Configure swappiness for Kubernetes workloads
   echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   ```

3. **Verify Partition Alignment** (particularly important for NVMe):
   ```bash
   # Check partition alignment
   sudo blockdev --getalignoff /dev/nvme0n1p{1..7}
   ```
   All partitions should return "0" indicating proper alignment for NVMe performance

#### Partitioning Verification

To verify your partitioning is optimal for Kubernetes on the SER8:

```bash
# Check mount options
findmnt -t ext4,vfat

# Verify I/O scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Check partition alignment and block sizes
sudo blockdev --getbsz /dev/nvme0n1p{3..6}

# Check filesystem features
sudo tune2fs -l /dev/nvme0n1p5 | grep "Filesystem features"
```

This configuration is tailored for the Beelink SER8's NVMe storage and optimized for Kubernetes and Docker workloads, ensuring optimal performance for your K3s cluster.

## 4. Setting Up a User on Ser8

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

## 5. Creating SSH Key-Pair from Remote Machine

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

## 6. K3s Installation

For K3s installation on Omakub OS, we'll use Ansible. Please refer to the `ansible/` directory for detailed instructions and playbooks.

## Additional Resources

### Managing Omakub OS

Omakub OS includes several tools to help manage your system:

1. System updates:
   ```bash
   sudo omakub-update
   ```

2. Package management:
   ```bash
   sudo omakub-pkg install <package-name>
   sudo omakub-pkg remove <package-name>
   sudo omakub-pkg list
   ```

3. System monitoring:
   ```bash
   omakub-monitor
   ```

4. Configure Omakub to work with your K3s installation:
   ```bash
   sudo omakub setup --k3s-kubeconfig /etc/rancher/k3s/k3s.yaml
   ```

### Troubleshooting

If you encounter any issues during the setup process, check the following:
- System logs: `journalctl -xe`
- K3s logs: `sudo journalctl -u k3s`
- Omakub logs: `sudo journalctl -u omakub`
- Omakub system diagnostics: `sudo omakub-diag`
