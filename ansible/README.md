# Ansible Setup for K3s and Docker on Ser8

This directory contains Ansible playbooks and roles for setting up K3s and Docker on your Ser8 device. The setup allows for remote Docker usage from IntelliJ IDEA.

## Directory Structure

```
ansible/
├── inventory.yml           # Host definitions
├── k3s-docker-setup.yml    # Main playbook
├── roles/
│   ├── k3s/               # K3s installation and configuration
│   ├── docker/            # Docker installation and configuration
│   └── remote-docker/     # Setup for remote Docker connections
└── README.md              # This file
```

## Prerequisites

1. Ansible installed on your control machine:
   ```bash
   # For Ubuntu/Debian
   sudo apt update
   sudo apt install ansible

   # For macOS (using Homebrew)
   brew install ansible
   ```

2. SSH access to your Ser8 device configured as described in the main README.md

## Setup Instructions

### 1. Configure the Inventory

Edit the `inventory.yml` file to match your Ser8 device's IP address or hostname:

```yaml
all:
  hosts:
    ser8:
      ansible_host: your_ser8_ip_or_hostname
      ansible_user: k3sadmin
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

### 2. Run the Playbook

Execute the main playbook to install and configure K3s and Docker:

```bash
ansible-playbook -i inventory.yml k3s-docker-setup.yml
```

This playbook will:
- Install K3s as a single-node Kubernetes cluster
- Install Docker and configure it for remote access
- Set up TLS certificates for secure Docker remote connections
- Configure necessary firewall rules

### 3. Connect IntelliJ IDEA to Remote Docker

After running the playbook, follow these steps to connect IntelliJ IDEA to the remote Docker instance:

1. Copy the Docker client certificates from Ser8 to your local machine:
   ```bash
   mkdir -p ~/.docker/ser8
   scp k3sadmin@ser8-hostname:~/.docker/ca.pem ~/.docker/ser8/
   scp k3sadmin@ser8-hostname:~/.docker/cert.pem ~/.docker/ser8/
   scp k3sadmin@ser8-hostname:~/.docker/key.pem ~/.docker/ser8/
   ```

2. In IntelliJ IDEA:
   - Go to Settings/Preferences → Build, Execution, Deployment → Docker
   - Click on "+" to add a new Docker configuration
   - Select "TCP socket"
   - Enter `tcp://ser8-hostname:2376` as the Engine API URL
   - Check "TLS certificates" and specify the path to the certificates directory (~/.docker/ser8)
   - Apply and OK

3. Verify the connection in IntelliJ IDEA:
   - Open the Docker tool window (View → Tool Windows → Docker)
   - You should see your remote Docker daemon connected and any existing containers/images

## Customization

You can customize the installation by modifying the variables in:
- `roles/k3s/defaults/main.yml`
- `roles/docker/defaults/main.yml`
- `roles/remote-docker/defaults/main.yml`

## Troubleshooting

If you encounter issues:

1. Check connectivity:
   ```bash
   ansible -i inventory.yml all -m ping
   ```

2. Verify Docker remote connection:
   ```bash
   docker --tlsverify --tlscacert=~/.docker/ser8/ca.pem \
          --tlscert=~/.docker/ser8/cert.pem \
          --tlskey=~/.docker/ser8/key.pem \
          -H tcp://ser8-hostname:2376 info
   ```

3. Check service status on Ser8:
   ```bash
   ssh k3sadmin@ser8-hostname "sudo systemctl status k3s"
   ssh k3sadmin@ser8-hostname "sudo systemctl status docker"
   ```
