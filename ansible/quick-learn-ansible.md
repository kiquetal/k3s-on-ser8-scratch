# Quick Start Guide to Ansible for K3s and Docker Setup

This guide provides a quick introduction to Ansible fundamentals, explaining the components and structure used in this project.

## What is Ansible?

Ansible is an open-source automation tool that simplifies software provisioning, configuration management, and application deployment. It uses a declarative language to describe system configurations and a push-based mechanism to deploy them.

## Key Ansible Components

### 1. Inventory

**File:** `inventory.yml`

The inventory defines the hosts and groups that Ansible will manage. In our case, it contains the Ser8 device information.

```yaml
all:
  hosts:
    ser8:
      ansible_host: your_ser8_ip_or_hostname
      ansible_user: k3sadmin
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

- `ansible_host`: The hostname or IP address of your Ser8 device
- `ansible_user`: The SSH user to connect with
- `ansible_ssh_private_key_file`: Path to your SSH private key

### 2. Playbook

**File:** `k3s-docker-setup.yml`

Playbooks are YAML files that define a set of tasks to be executed on hosts. Our main playbook includes:

```yaml
- name: Set up K3s and Docker for remote access
  hosts: ser8
  become: true
  roles:
    - role: docker
    - role: remote-docker
    - role: k3s
```

- `name`: Description of what the playbook does
- `hosts`: Specifies which hosts from the inventory to target
- `become: true`: Enables privilege escalation (sudo)
- `roles`: List of roles to apply

### 3. Roles

Roles are a way to organize related tasks, variables, and files. Our project has three roles:

#### 3.1 Docker Role (`roles/docker/`)

Installs and configures Docker on the Ser8 device.

- `defaults/main.yml`: Default variables for Docker installation
- `handlers/main.yml`: Handlers for restarting services
- `tasks/main.yml`: Tasks for installing and configuring Docker
- `templates/daemon.json.j2`: Jinja2 template for Docker daemon configuration

#### 3.2 K3s Role (`roles/k3s/`)

Installs and configures K3s on the Ser8 device.

- `defaults/main.yml`: Default variables for K3s installation
- `tasks/main.yml`: Tasks for installing and configuring K3s

#### 3.3 Remote Docker Role (`roles/remote-docker/`)

Configures Docker for remote access from IntelliJ IDEA.

- `defaults/main.yml`: Default variables for remote Docker configuration
- `tasks/main.yml`: Tasks for setting up TLS certificates
- `templates/docker-config.json.j2`: Template for Docker client configuration

## Role Directory Structure

Each role follows a standard directory structure:

```
role_name/
├── defaults/       # Default variables (lowest precedence)
│   └── main.yml
├── files/          # Static files to be copied to the managed hosts
├── handlers/       # Handlers to restart services or trigger actions
│   └── main.yml
├── tasks/          # Main list of tasks to be executed
│   └── main.yml
└── templates/      # Jinja2 templates to be rendered
    └── *.j2
```

## How to Write Ansible Tasks

Ansible tasks follow a YAML format. Each task includes:

1. A name (optional but recommended)
2. A module name (e.g., `apt`, `template`, `file`)
3. Module parameters

Example:

```yaml
- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
    state: present
```

## Variables and Templates

### Variables

Variables in Ansible can be defined at multiple levels:

- In `defaults/main.yml` (lowest precedence)
- In the playbook
- On the command line using `--extra-vars`

Example from our Docker role:

```yaml
docker_version: "latest"
docker_users:
  - "{{ ansible_user }}"
docker_daemon_options:
  hosts:
    - "unix:///var/run/docker.sock"
    - "tcp://0.0.0.0:2376"
```

### Templates (Jinja2)

Templates use the Jinja2 templating language to dynamically generate files:

```json
{
  "hosts": {{ docker_daemon_options.hosts | to_json }},
  "tls": {{ docker_daemon_options.tls | to_json }},
  "tlsverify": {{ docker_daemon_options.tlsverify | to_json }}
}
```

## Handlers

Handlers are tasks that only run when notified by another task. They are typically used to restart services:

```yaml
- name: restart docker
  systemd:
    name: docker
    state: restarted
```

To trigger a handler:

```yaml
- name: Configure Docker daemon
  template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
  notify: restart docker
```

## Running Ansible

Basic command to run a playbook:

```bash
ansible-playbook -i inventory.yml k3s-docker-setup.yml
```

To run specific tasks using tags:

```bash
ansible-playbook -i inventory.yml k3s-docker-setup.yml --tags docker,remote
```

## Best Practices

1. Use meaningful names for tasks
2. Group related tasks into roles
3. Use variables for values that might change
4. Use handlers for service restarts
5. Test playbooks with `--check` flag before applying

## Learning More

To learn more about Ansible:

1. Official documentation: [https://docs.ansible.com](https://docs.ansible.com)
2. Ansible Galaxy (for community roles): [https://galaxy.ansible.com](https://galaxy.ansible.com)
3. Red Hat Ansible Blog: [https://www.ansible.com/blog](https://www.ansible.com/blog)

## Specific Components in this Project

### Docker Installation

The Docker role installs Docker using the official Docker repository, configures it for remote access, and sets up users.

### K3s Installation

The K3s role installs K3s using the official installation script with customizable options.

### Remote Docker Access

The remote-docker role generates TLS certificates for secure communication between IntelliJ IDEA and Docker on Ser8.

## Common Ansible Modules Used in This Project

- `apt`: Manages packages on Debian-based systems like Omakub OS
- `template`: Renders Jinja2 templates to configuration files
- `file`: Creates directories and manages file permissions
- `systemd`: Controls system services
- `openssl_privatekey` and `openssl_certificate`: Manages SSL/TLS certificates
- `command` and `shell`: Runs commands on the managed host
