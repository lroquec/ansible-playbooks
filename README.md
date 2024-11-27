# Ansible Playbook: System Setup

This playbook automates system configuration tasks for Red Hat and Ubuntu-based systems. It ensures the system is updated, installs required packages, configures the firewall, creates a user with `sudo` privileges, and disables root SSH login.

## Features

1. Updates the system packages.
2. Installs a list of packages (common and OS-specific).
3. Configures and enables the firewall:
   - Opens specified ports.
4. Creates a new user (`sysadm` by default) with `sudo` privileges.
5. Disables SSH login for the `root` user.

## Prerequisites

- Ansible installed on the control machine.
- Target hosts configured in an inventory file.
- SSH access to the target systems with appropriate permissions.

## Supported Operating Systems

- Red Hat-based systems (RHEL, CentOS, etc.).
- Debian-based systems (Ubuntu, etc.).

## Usage

### Inventory File

Create an inventory file (e.g., `inventory`) with the target hosts:

```ini
[all]
server1 ansible_host=192.168.1.10 ansible_user=root
server2 ansible_host=192.168.1.11 ansible_user=root
```
 ### Execute the Playbook

```shell
 ansible-playbook -i inventory site.yml --extra-vars "new_user_password=YoursuPerStrongPassword!"
```