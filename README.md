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

### 3.2 Standard Linux Partitioning for Kubernetes

To create an optimal partition layout for Kubernetes using standard Linux tools:

1. **Access a Terminal**:
   - During installation, access a terminal using `Ctrl+Alt+F2`
   - You may need to log in as the installer user (often "root" with no password)

2. **Identify Target Disk**:
   ```bash
   # List available disks and identify your target installation disk
   lsblk -o NAME,SIZE,MODEL,TRAN
   fdisk -l
   ```

3. **Create Partitions Using fdisk**:
   ```bash
   # Start fdisk for your target disk (replace /dev/sda with your actual disk)
   sudo fdisk /dev/sda
   ```

   In the fdisk interactive prompt:
   ```
   # Create new GPT partition table
   g

   # Create EFI partition
   n
   1
   [press Enter for default]
   +512M
   t
   1
   ef

   # Create boot partition
   n
   2
   [press Enter for default]
   +1G
   
   # Create root partition
   n
   3
   [press Enter for default]
   +40G

   # Create /var partition
   n
   4
   [press Enter for default]
   +30G

   # Create docker storage partition
   n
   5
   [press Enter for default]
   +80G

   # Create k3s data partition
   n
   6
   [press Enter for default]
   +40G

   # Create swap partition
   n
   7
   [press Enter for default]
   +8G
   t
   7
   82

   # Write changes and exit
   w
   ```

4. **Format Partitions**:
   ```bash
   # Format EFI partition
   sudo mkfs.fat -F32 /dev/sda1

   # Format boot partition
   sudo mkfs.ext4 -L boot /dev/sda2

   # Format root partition with optimizations for SSD/NVMe
   sudo mkfs.ext4 -L root -O fast_commit,extent,dir_index /dev/sda3

   # Format var partition
   sudo mkfs.ext4 -L var -O fast_commit,extent,dir_index /dev/sda4

   # Format docker storage with optimizations for containers
   sudo mkfs.ext4 -L docker -i 8192 -I 256 -O fast_commit,extent,dir_index,64bit /dev/sda5

   # Format k3s partition
   sudo mkfs.ext4 -L k3s -O fast_commit,extent,dir_index /dev/sda6

   # Set up swap
   sudo mkswap -L swap /dev/sda7
   ```

5. **Mount for Installation**:
   ```bash
   # Mount root filesystem
   sudo mount /dev/sda3 /mnt

   # Create mount points
   sudo mkdir -p /mnt/{boot/efi,var,var/lib/docker,var/lib/rancher}

   # Mount other partitions
   sudo mount /dev/sda1 /mnt/boot/efi
   sudo mount /dev/sda2 /mnt/boot
   sudo mount /dev/sda4 /mnt/var
   sudo mount /dev/sda5 /mnt/var/lib/docker
   sudo mount /dev/sda6 /mnt/var/lib/rancher
   ```

6. **Generate fstab for Optimized Mounts**:
   ```bash
   sudo mkdir -p /mnt/etc
   sudo bash -c 'echo "# /etc/fstab: system mounts" > /mnt/etc/fstab'
   
   # Add entries with optimal mount options
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda1) /boot/efi vfat defaults 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda2) /boot ext4 defaults,noatime 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda3) / ext4 defaults,noatime,discard=async 0 1" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda4) /var ext4 defaults,noatime,discard=async 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda5) /var/lib/docker ext4 defaults,noatime,discard=async,nobarrier 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda6) /var/lib/rancher ext4 defaults,noatime,discard=async 0 2" | sudo tee -a /mnt/etc/fstab
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda7) none swap sw 0 0" | sudo tee -a /mnt/etc/fstab
   ```

7. **Continue Installation**:
   - Return to the graphical installer with `Ctrl+Alt+F7`
   - Direct the installer to use the mount points you've created
   - Continue with the rest of the installation process

#### Verification of Partition Layout

After partitioning, verify your setup with:

```bash
# Check partition table
sudo fdisk -l /dev/sda

# Check filesystem types
lsblk -f

# Check mount options (after installation is complete)
findmnt -t ext4,vfat
```

### 3.3 Disk Encryption Setup

For enhanced security, you can encrypt sensitive partitions:

1. **Full LUKS Encryption** (apply before formatting):
   ```bash
   # Encrypt the root partition
   sudo cryptsetup luksFormat /dev/sda3
   
   # Open the encrypted partition
   sudo cryptsetup open /dev/sda3 cryptroot
   
   # Format the opened LUKS container
   sudo mkfs.ext4 -L root /dev/mapper/cryptroot
   
   # Mount for installation
   sudo mount /dev/mapper/cryptroot /mnt
   ```

2. **Selective Partition Encryption**:
   ```bash
   # Encrypt Docker storage
   sudo cryptsetup luksFormat /dev/sda5
   sudo cryptsetup open /dev/sda5 cryptdocker
   sudo mkfs.ext4 -L docker /dev/mapper/cryptdocker
   sudo mount /dev/mapper/cryptdocker /mnt/var/lib/docker
   
   # Encrypt K3s data
   sudo cryptsetup luksFormat /dev/sda6
   sudo cryptsetup open /dev/sda6 cryptk3s
   sudo mkfs.ext4 -L k3s /dev/mapper/cryptk3s
   sudo mount /dev/mapper/cryptk3s /mnt/var/lib/rancher
   ```

3. **Performance Considerations**:
   - Encryption adds a small overhead, especially with mechanical HDDs
   - Modern CPUs with AES-NI support minimize this impact on SSDs
   - For optimal performance, consider encrypting only sensitive partitions

4. **Configure crypttab** (if using encryption):
   ```bash
   # Create crypttab file
   sudo bash -c 'echo "# /etc/crypttab: mappings for encrypted partitions" > /mnt/etc/crypttab'
   
   # Add entries for encrypted partitions
   echo "cryptroot UUID=$(sudo blkid -s UUID -o value /dev/sda3) none luks" | sudo tee -a /mnt/etc/crypttab
   echo "cryptdocker UUID=$(sudo blkid -s UUID -o value /dev/sda5) none luks" | sudo tee -a /mnt/etc/crypttab
   echo "cryptk3s UUID=$(sudo blkid -s UUID -o value /dev/sda6) none luks" | sudo tee -a /mnt/etc/crypttab
   ```

Continue with the installation process after completing the partitioning.

### 3.4 Modern Partitioning for Beelink SER8

The Beelink SER8 typically comes with NVMe storage (1TB in this configuration) and 24GB of RAM, which requires specific partitioning approaches. Here's how to create the optimized partition layout using standard Linux tools:

#### Using parted for Modern NVMe Partitioning

1. **Boot from Installation Media and Access Terminal**:
   After booting from your installation media, access a terminal by pressing `Ctrl+Alt+F2`.

2. **Identify your NVMe drive**:
   ```bash
   lsblk -o NAME,SIZE,TRAN,MODEL
   ```
   
   Typically, NVMe drives appear as `/dev/nvme0n1`

3. **Create a GPT partition table**:
   ```bash
   sudo parted /dev/nvme0n1 mklabel gpt
   ```

4. **Create the partitions using parted** (adjusted for SER8's 1TB NVMe and 24GB RAM):
   ```bash
   # Create EFI partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "EFI" fat32 0% 512MiB
   sudo parted /dev/nvme0n1 set 1 esp on
   
   # Create boot partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "boot" ext4 512MiB 2GiB
   
   # Create root partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "root" ext4 2GiB 82GiB
   
   # Create var partition
   sudo parted -a optimal /dev/nvme0n1 mkpart "var" ext4 82GiB 152GiB
   
   # Create docker storage partition (larger for container images)
   sudo parted -a optimal /dev/nvme0n1 mkpart "docker" ext4 152GiB 552GiB
   
   # Create k3s data partition (larger for Kubernetes workloads)
   sudo parted -a optimal /dev/nvme0n1 mkpart "k3s" ext4 552GiB 988GiB
   
   # Create swap partition (adjusted for 24GB RAM - using 12GB)
   sudo parted -a optimal /dev/nvme0n1 mkpart "swap" linux-swap 988GiB 100%
   ```

   > **Note about partition sizes**: With a 1TB NVMe drive, we allocate more space to Docker (400GB) and K3s (436GB) partitions to accommodate large container images and Kubernetes workloads. The swap is sized to 12GB (half of RAM) which is appropriate for the SER8's 24GB RAM.

5. **Verify the partition layout**:
   ```bash
   sudo parted /dev/nvme0n1 print
   ```

6. **Format the partitions with optimizations for NVMe**:
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
   ```

7. **Mount partitions for installation**:
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

8. **Generate optimized fstab**:
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

9. **Continue with your standard installation process**:
   Return to the graphical installer by pressing `Ctrl+Alt+F7` or continue with your text-based installation, pointing it to use the mount points you've created.

#### SER8-Specific Optimization

After installation, apply these optimizations specifically designed for the Beelink SER8's hardware:

1. **I/O Scheduler Configuration**:
   ```bash
   # Set the scheduler to none for NVMe (best performance on SER8)
   echo 'ACTION=="add|change", KERNEL=="nvme0n1", ATTR{queue/scheduler}="none"' | sudo tee /etc/udev/rules.d/60-schedulers.rules
   ```

2. **Memory and Swap Optimization for Kubernetes** (adjusted for 24GB RAM):
   ```bash
   # Configure swappiness for Kubernetes workloads
   echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   
   # Adjust the kernel's memory management for larger RAM
   echo 'vm.min_free_kbytes=262144' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   echo 'vm.zone_reclaim_mode=0' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   ```

3. **Enable and configure the swap**:
   ```bash
   # Enable the swap
   sudo swapon /dev/nvme0n1p7
   
   # Optimize swap performance
   echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-sysctl.conf
   ```

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
