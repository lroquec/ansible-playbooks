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
server1 ansible_host=192.168.1.10 ansible_user=sysadm
server2 ansible_host=192.168.1.11 ansible_user=sysadm
```
 ### Execute the Playbook

```shell
 ansible-playbook -i inventory site.yml --extra-vars "new_user_password=YoursuPerStrongPassword!"
```
---

# Kubernetes Cluster Deployment with Ansible

This repository contains Ansible playbooks to deploy a Kubernetes cluster using kubeadm. The playbooks are designed to be reusable and configurable using variables.

## Playbooks

### 1. `k8s_prerequisites.yaml`
**Description**: 
This playbook prepares the nodes (Control Plane and Workers) for Kubernetes installation.

**Key tasks**:
- Updates and upgrades system packages.
- Installs required dependencies like `curl`, `socat`, etc.
- Disables `swap` on the system.
- Configures required kernel parameters (`sysctl`).
- Loads necessary kernel modules for Kubernetes (`overlay`, `br_netfilter`).

**Key variables**:
- None specific, as it applies generic configurations to all nodes.

---

### 2. `k8s_init_all.yaml`
**Description**: 
This playbook initializes all nodes with Kubernetes packages.

**Key tasks**:
- Configures the Kubernetes repository.
- Installs essential components (`kubeadm`, `kubectl`, `kubelet`) at a specified version.
- Creates and uses the `kubeadm-config.yaml` file to initialize the cluster.
- Sets up the node alias in `/etc/hosts` to facilitate communication.
- Executes the `kubeadm init` command to initialize the cluster.

**Key variables**:
- `kubernetes_repo_version`: Kubernetes repository version (e.g., `v1.31`).
- `kubernetes_package_version`: Version of Kubernetes packages (e.g., `1.31.1-1.1`).
- `control_plane_alias`: Alias for the Control Plane node.
- `control_plane_ip`: IP address of the Control Plane node.

---

### 3. `k8s_init_controlplane.yaml`
**Description**: 
This Ansible playbook is designed to initialize the Kubernetes control plane on a set of hosts. It performs the following tasks:

1. Checks if the Kubernetes cluster is already initialized by checking for the existence of the `/etc/kubernetes/admin.conf` file.
2. If the cluster is not initialized, it runs the `kubeadm init` command with the specified `pod_network_cidr` (Pod Network CIDR) value.
3. Saves the output of the `kubeadm init` command to `/root/kubeadm-init.out` file.
4. Displays the output of the `kubeadm init` command.
5. Creates the `.kube` directory for the root user with appropriate permissions.
6. Copies the `admin.conf` file to the root user's `.kube/config` file.
7. Creates the `.kube` directory for a specified user (`username` variable) with appropriate permissions.
8. Copies the `admin.conf` file to the specified user's `.kube/config` file.
9. Sets up kubectl autocomplete and k alias.

#### Variables

The playbook uses the following variables:

- `pod_network_cidr`: The CIDR notation for the Pod Network. Default value: `"10.244.0.0/16"`.
- `username`: The name of the user for whom the `.kube` directory and configuration file will be created.

---

### 4. `k8s_add_worker.yml`
**Description**: 
This playbook adds **Worker** nodes to the cluster using the token generated on the Control Plane node.

**Key tasks**:
- Uses the token and hash generated on the Control Plane to join Worker nodes to the cluster.
- Configures the name of the Worker node.

**Key variables**:
- `control_plane_ip`: IP address of the Control Plane node.
- `token`: Token generated by `kubeadm`.
- `discovery_hash`: Certificate hash generated by `kubeadm`.
- `node_name`: Name of the Worker node.

---

## How to Use

1. **Prepare the nodes**:
   Run the `k8s_prerequisites.yaml` playbook on all nodes:
   ```bash
   ansible-playbook -i inventory.yml k8s_prerequisites.yaml
   ```
   Run the `k8s_init_all.yaml` playbook on all nodes:
   ```bash
   ansible-playbook -i inventory.yml k8s_init_all.yaml
   ```   
2. **Initialize the Control Plane: Run the k8s_init_controlplane.yaml playbook on the Control Plane node**:
   1. Ensure that the target hosts are accessible via SSH and have the necessary prerequisites installed (Ansible, Kubernetes, etc.).
   2. Modify the `hosts` section of the playbook to specify the target hosts where the control plane should be initialized.
   3. Review and adjust the `pod_network_cidr` and `username` variables as needed.
   4. Run the playbook using the following command:
   ```bash
   ansible-playbook -i inventory.yml k8s_init_controlplane.yaml
   ```
3. **Add Worker Nodes: Run the add_worker_nodes.yml playbook on the Worker nodes**:
   With the join info from previous playbook run
   ```bash
   ansible-playbook -i inventory.yml add_worker_nodes.yml
   ```
4. **Add network plugin**:
   ```bash
   ansible-playbook -i inventory.yml k8s_add_network.yaml
   ```
5. **Add ingress controller if needed**:
   ```bash
   ansible-playbook -i inventory.yml k8s_add_ingress.yaml
   ```

# Kubernetes Master Node Update Playbook

This playbook automates the update of a Kubernetes master node using specific versions of `kubeadm`, `kubelet`, and `kubectl`. The process ensures a consistent and locked Kubernetes version after the update.

## Variables
- `kube_repo_version`: Specifies the desired Kubernetes repository version (e.g., "1.31").
- `kube_version`: Specifies the desired Kubernetes version (e.g., "1.31.1").

## Playbook Tasks
1. **Update the Kubernetes Repository**  
   Adds the specified Kubernetes repository.

2. **Update Apt Cache**  
   Updates the package manager cache.

3. **Unlock the `kubeadm` Package**  
   Ensures the `kubeadm` package is not held back.

4. **Install a Specific Version of `kubeadm`**  
   Installs the version defined in `kube_version`.

5. **Lock the `kubeadm` Package**  
   Holds the `kubeadm` package at the installed version.

6. **Plan the Cluster Upgrade**  
   Runs `kubeadm upgrade plan` to preview the upgrade process.

7. **Apply the Cluster Upgrade**  
   Executes `kubeadm upgrade apply` to upgrade the cluster if the version is supported.

8. **Unlock `kubelet` and `kubectl` Packages**  
   Removes the hold on the `kubelet` and `kubectl` packages.

9. **Install Specific Versions of `kubelet` and `kubectl`**  
   Installs the versions defined in `kube_version`.

10. **Lock `kubelet` and `kubectl` Packages**  
    Holds the `kubelet` and `kubectl` packages at the installed versions.

11. **Reload the System Daemon**  
    Reloads the system daemon for changes to take effect.

12. **Restart the `kubelet` Service**  
    Restarts the `kubelet` service to apply updates.

## Usage
To execute the playbook, run the following command:

```bash
ansible-playbook -i <inventory_file> update_kubernetes_master.yml
```
Replace <inventory_file> with your Ansible inventory file path.

# Kubernetes Worker Nodes Update Playbook

This playbook automates the update process for Kubernetes worker nodes. It ensures that specific versions of `kubeadm`, `kubelet`, and `kubectl` are installed, locked, and properly configured for consistency across the cluster.

## Variables
- `kube_repo_version`: Specifies the desired Kubernetes repository version (e.g., "1.31").
- `kube_version`: Specifies the desired Kubernetes version (e.g., "1.31.3").

## Playbook Tasks
1. **Update the Kubernetes Repository**  
   Adds the specified Kubernetes repository to the system.

2. **Update Apt Cache**  
   Updates the package manager cache to reflect the new repository.

3. **Unlock the `kubeadm` Package**  
   Ensures the `kubeadm` package is not held back.

4. **Install a Specific Version of `kubeadm`**  
   Installs the version defined in `kube_version`.

5. **Lock the `kubeadm` Package**  
   Holds the `kubeadm` package at the installed version.

6. **Upgrade the Node Configuration**  
   Runs `kubeadm upgrade node` to apply node-level configuration updates.

7. **Unlock `kubelet` and `kubectl` Packages**  
   Removes the hold on the `kubelet` and `kubectl` packages.

8. **Install Specific Versions of `kubelet` and `kubectl`**  
   Installs the versions defined in `kube_version`.

9. **Lock `kubelet` and `kubectl` Packages**  
   Holds the `kubelet` and `kubectl` packages at the installed versions.

10. **Reload the System Daemon**  
    Reloads the system daemon to ensure all configuration changes are applied.

11. **Restart the `kubelet` Service**  
    Restarts the `kubelet` service to apply updates.

## Usage
To execute the playbook, run the following command:

```bash
ansible-playbook -i <inventory_file> update_kubernetes_worker.yml
```
Replace <inventory_file> with your Ansible inventory file path.

---

# Add Ingress NGINX Manifest Playbook

An Ansible playbook to deploy the Ingress NGINX controller on Kubernetes master nodes. This playbook applies the Ingress NGINX manifest to your Kubernetes cluster, ensuring that the necessary Ingress controller is installed and configured properly.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Playbook Overview](#playbook-overview)
- [Variables](#variables)
- [Tasks](#tasks)
- [Usage](#usage)
- [Example](#example)
- [License](#license)
- [Contributing](#contributing)

## Prerequisites

Before running this playbook, ensure that the following prerequisites are met:

- **Ansible** is installed on the control node.
- **kubectl** is installed and configured on the target master nodes.
- **Kubernetes Cluster** is up and running with designated master nodes.
- **Appropriate Permissions** to execute commands on the master nodes (e.g., SSH access).
- **Internet Access** on master nodes to fetch the Ingress NGINX manifest.

## Playbook Overview

This playbook performs the following actions:

1. **Applies the Ingress NGINX Manifest**: Downloads and applies the Ingress NGINX controller manifest to the Kubernetes cluster.
2. **Retries on Failure**: Attempts to apply the manifest up to 5 times with a 10-second delay between retries in case of failures.
3. **Error Handling**: Continues execution even if the manifest application fails, allowing for debugging.
4. **Debug Message**: Outputs a success message if the Ingress NGINX is applied successfully.

## Variables

### `ingress_nginx_manifest`

- **Description**: URL to the Ingress NGINX controller manifest YAML file.
- **Default Value**: 
  ```yaml
  "https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/baremetal/deploy.yaml"

---

# Ansible Playbook: Install and Patch Metrics Server

## Overview

This Ansible playbook installs the Metrics Server in a Kubernetes cluster and patches its deployment to include the `--kubelet-insecure-tls` argument. This ensures compatibility with clusters where kubelet's certificate validation is not configured.

## Features

The playbook performs the following tasks:
1. Installs the Metrics Server components using the official manifest from the Kubernetes GitHub repository.
2. Waits for the Metrics Server deployment to be successfully created in the `kube-system` namespace.
3. Applies a JSON patch to add the `--kubelet-insecure-tls` argument to the Metrics Server container.
4. Verifies that the patch has been applied successfully.

## Prerequisites

1. **Kubernetes Cluster**: Ensure you have a running Kubernetes cluster with kubeadm installed.
2. **kubectl**: The `kubectl` command-line tool must be available on the Ansible control node and configured to connect to the cluster.
3. **Ansible**: Ansible 2.9 or later should be installed on the control node.
4. **Target Host(s)**: The playbook should target the Kubernetes master node(s).

## Playbook Variables

| Variable                | Description                                                                                     | Default Value                                                                                  |
|-------------------------|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| `metrics_server_manifest` | URL of the Metrics Server components YAML file to be applied.                                 | `https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` |
| `metrics_server_patch`    | JSON patch to add the `--kubelet-insecure-tls` argument to the Metrics Server deployment.      | `[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]` |

## How to Use

1. Clone this repository or save the playbook file locally.
2. Update your Ansible inventory file to include your Kubernetes master nodes under the `k8s_masters` group.
3. Run the playbook with the following command:
   ```bash
   ansible-playbook -i inventory metrics_server_install_patch.yaml
   ```
---

# Argo Projects Ansible Playbook

This Ansible playbook is designed to automate the deployment of Argo CD, Argo Rollouts, and Argo CD Image Updater to a Kubernetes cluster.

## Prerequisites

- A Kubernetes cluster is already set up and accessible.
- `kubectl` is installed and configured to interact with your cluster.
- Ansible is installed on the control node.

## Playbook Overview

### Hosts
The playbook is executed on the `controlplane`.

### Variables
- `argocd_manifest`: URL for the Argo CD installation manifest.
- `argorollouts_manifest`: URL for the Argo Rollouts installation manifest.
- `argocd_image_updater_manifest`: URL for the Argo CD Image Updater installation manifest.

### Tasks
1. **Add `argocd` Namespace**: Creates the namespace for Argo CD.
2. **Apply Argo CD Manifest**: Deploys Argo CD components to the `argocd` namespace.
3. **Verify Argo CD Deployment**: Confirms the successful application of the Argo CD manifest.
4. **Apply Argo CD Image Updater Manifest**: Deploys the Argo CD Image Updater to the `argocd` namespace.
5. **Verify Argo CD Image Updater Deployment**: Confirms the successful application of the Argo CD Image Updater manifest.
6. **Add `argo-rollouts` Namespace**: Creates the namespace for Argo Rollouts.
7. **Apply Argo Rollouts Manifest**: Deploys Argo Rollouts to the `argo-rollouts` namespace.
8. **Verify Argo Rollouts Deployment**: Confirms the successful application of the Argo Rollouts manifest.

## Usage

1. **Clone or Download** this repository and ensure you have the `k8s_add_argocd.yaml` file locally.
2. **Check Ansible Configuration**: Confirm your `hosts` file or inventory is set to run against the `controlplane` host.
3. **Execute the Playbook**:
   ```bash
   ansible-playbook -i inventory.ini tests/ansible-playbooks/k8s/k8s_add_argo_projects.yaml
   ```

Here's a markdown README for the Kubernetes Dashboard installation playbook:

# Kubernetes Dashboard Installation Playbook

This Ansible playbook automates the installation and configuration of the Kubernetes Dashboard using Helm. It sets up a complete dashboard environment with proper authentication and RBAC configuration.

## Prerequisites

- Kubernetes cluster up and running
- Ansible installed on the control machine
- `kubectl` configured with cluster access
- Helm installed on the control machine

## Features

- Installs Kubernetes Dashboard using Helm
- Creates dedicated namespace
- Configures service account with admin privileges
- Sets up authentication token
- Enables metrics scraper
- Configures RBAC permissions

## Variables

You can customize the installation by modifying the following variables in the playbook:

```yaml
helm_chart_version: "7.10.0"
release_name: kubernetes-dashboard
namespace: kubernetes-dashboard
service_type: ClusterIP
```

## Installation

1. Clone this repository
2. Review and adjust variables if needed
3. Run the playbook:

```bash
ansible-playbook install-kubernetes-dashboard.yml
```

## Post-Installation

After successful installation, the playbook will display:
- The access token for dashboard authentication
- Instructions to access the dashboard

### Accessing the Dashboard

1. Run the port-forward command:
```bash
kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard-kong-proxy 8000:443
```

2. Access the dashboard through your browser:
```
https://localhost:8000
```

3. Use the token displayed during installation to log in

## Security Considerations

- The playbook creates an admin user with cluster-admin privileges
- Token TTL is set to 0 (tokens don't expire)
- Service type is set to ClusterIP by default for security
- HTTPS is enabled by default

## Components Installed

- Kubernetes Dashboard
- Metrics Scraper
- Required RBAC configurations
- ServiceAccount and ClusterRoleBinding
- Authentication token secret

## Troubleshooting

If you encounter issues:

1. Verify all prerequisites are met
2. Check if the namespace was created successfully
3. Ensure Helm repositories are accessible
4. Verify the token secret was created properly

## Notes

- The dashboard is configured with metrics scraper enabled
- The metrics-server is disabled by default
- The service is configured as ClusterIP for security
- Token-based authentication is implemented

## Requirements

The playbook will install the following packages:
- python3-kubernetes
- python3-openshift
- python3-yaml
- python3-pip
- curl
- apt-transport-https

# Kube-Prometheus-Stack Installation with Ansible

This repository contains Ansible playbooks for deploying the kube-prometheus-stack Helm chart with persistent storage for Prometheus and Grafana.

## Prerequisites

- Kubernetes cluster with a configured StorageClass
- kubectl configured with access to your cluster
- Ansible 2.9 or higher
- Helm 3.x installed on your local machine

## Installation Steps

1. Install required Ansible collections:
```bash
ansible-galaxy collection install kubernetes.core
```

2. Configure your variables in the playbook:
   - Set your `storage_class_name` to match your cluster's storage class
   - Adjust `prometheus_storage_size` if needed (default: 50Gi)
   - Update `helm_chart_version` if you need a specific version

3. Run the playbook:
```bash
ansible-playbook playbook.yml
```

## Configuration Details

### Storage Configuration

- **Prometheus**: Uses a VolumeClaimTemplate for persistent storage
  - Default size: 10Gi
  - Access mode: ReadWriteOnce

- **Grafana**: Uses a pre-created PVC
  - Size: 10Gi
  - Access mode: ReadWriteOnce
  - PVC name: grafana-pvc

### Components

- Prometheus Server with persistent storage
- Grafana with persistent storage
- Node Exporter
- Kube State Metrics

### Data Persistence

#### Grafana Persistence
The playbook creates a dedicated PVC for Grafana that persists:
- User configurations
- Dashboards
- Data source settings
- Alert configurations
- User preferences

**Note**: After changing the Grafana admin password through the UI, it will persist across pod restarts and redeployments as long as the PVC is maintained.

#### Prometheus Persistence
Prometheus data is persisted using a VolumeClaimTemplate, ensuring:
- Metric history is preserved
- Queries and rules persist across restarts
- Data survives pod rescheduling

## Maintenance

### Updating the Installation

To update the installation:

1. Update the `helm_chart_version` in the playbook
2. Re-run the playbook:
```bash
ansible-playbook playbook.yml
```

### Uninstalling

To remove the installation:
```bash
helm uninstall prometheus -n monitoring
```

**Note**: This will not delete the Grafana PVC. To delete it:
```bash
kubectl delete pvc grafana-pvc -n monitoring
```

## Troubleshooting

1. Check pod status:
```bash
kubectl get pods -n monitoring
```

2. View Grafana logs:
```bash
kubectl logs -f deployment/prometheus-grafana -n monitoring
```

3. View Prometheus logs:
```bash
kubectl logs -f statefulset/prometheus-prometheus-kube-prometheus-prometheus -n monitoring
```

---

# Ansible Playbook: Install Helm

This Ansible playbook automates the installation of Helm, the Kubernetes package manager, using the official installation script.

## Features

- Downloads the official Helm installation script.
- Executes the script to install the latest version of Helm.
- Verifies the installation by checking the Helm version.

## Prerequisites

- Ansible installed on your local machine.
- SSH access to the target machines.
- `sudo` privileges for the remote user.
- Internet access on the target machines to download the Helm installation script.

## How to Use

1. Clone this repository or save the playbook file.
2. Ensure your inventory file (`inventory`) contains the target hosts for the installation.
3. Run the playbook with the following command:
   ```bash
   ansible-playbook -i inventory install_helm_script.yml
   ```
## Expected Output

- Helm will be installed on the target machines.
- The installed Helm version will be displayed at the end of the playbook execution.

## Notes

- The playbook uses the official Helm installation script hosted on GitHub.
- Make sure the target machines can access `https://raw.githubusercontent.com`.

---

# Comprehensive Server Security Hardening Playbook

## Overview

This Ansible playbook provides a robust, multi-layered security solution for Ubuntu and Alma Linux servers, focusing on intrusion prevention and system hardening.

### Key Features

- 🔒 Fail2Ban configuration
- 🛡️ Multi-vector attack prevention
- 🌐 Cross-distribution compatibility
- 🚫 Protection against:
  - SSH brute-force attacks
  - WordPress login attempts
  - HTTP errors
  - Web scrapers
  - SQL injection attempts

## Prerequisites

### Supported Distributions
- Ubuntu (Debian-based)
- Alma Linux (Red Hat-based)

### Requirements
- Ansible 2.9+
- Python 3
- Root/sudo access

## Installation

1. Clone the repository:
```bash
git clone git@github.com:lroquec/ansible-playbooks.git
cd ansible-playbooks
```

2. Install required Ansible collections:
```bash
ansible-galaxy collection install ansible.posix community.general
```

## Configuration

### Variables Overview

Edit `group_vars/all.yml` or pass variables during playbook execution:

#### Security Configuration
```yaml
security_config:
  enable_fail2ban: true        # Enable/disable Fail2Ban
  enable_modsecurity: false    # Enable/disable ModSecurity
```

#### Fail2Ban Jail Configuration
```yaml
fail2ban_jails:
  ssh:
    enabled: true
    maxretry: 3
    bantime: 3600  # 1 hour ban
  wordpress:
    enabled: true
    maxretry: 5
    bantime: 86400  # 1 day ban
```

### Customization Options

- Adjust `maxretry` to control attempt limits
- Modify `bantime` to set ban duration
- Enable/disable specific protection modules
- Customize log paths and filtering

## Usage

### Basic Execution
```bash
ansible-playbook -i inventory security_hardening.yml
```

### Selective Execution
```bash
# Run only Fail2Ban configuration
ansible-playbook -i inventory security_hardening.yml --tags fail2ban

# Skip ModSecurity installation
ansible-playbook -i inventory security_hardening.yml --skip-tags modsecurity
```

## Security Levels

### 🟢 Low Risk Configuration
- SSH protection
- Basic WordPress login prevention

### 🟠 Medium Risk Configuration
- Add HTTP error and scraper protection
- Moderate ban times

### 🔴 High Risk Configuration
- Full protection suite
- Long ban times
- ModSecurity integration

## Logging and Monitoring

- Security events logged to `/var/log/ansible-security-audit.log`
- Fail2Ban maintains its own logs for banned IPs
- Optional email alerting configurable

## Best Practices

1. Always test in a staging environment
2. Regularly update the playbook
3. Monitor system logs
4. Adjust configurations based on your specific needs

## Troubleshooting

### Common Issues
- Verify Ansible and Python versions
- Check log files for detailed error information
- Ensure proper SSH and sudo access

### Debugging
```bash
# Check Fail2Ban status
sudo systemctl status fail2ban

# View Fail2Ban logs
sudo tail -f /var/log/fail2ban.log
```

---

**Note**: Continuous security is a journey, not a destination. Regularly review and update your security configurations.