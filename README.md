# talos_proxmox Ansible Role

This Ansible role automates the provisioning and configuration of a Talos-based Kubernetes cluster on Proxmox.

## Features
- Automated VM provisioning on Proxmox
- Talos OS configuration and bootstrapping
- Multus CNI installation and VLAN network setup (optional)
- Configuration of static kubernetes resources (for example secrets)
- ArgoCD installation and app-of-apps deployment

## Requirements

- Ansible >= 2.12
- Collections:
   - community.general
   - community.proxmox
   - kubernetes.core
- Python >= 3.9 on the control host
- Python packages: `requests`, `kubernetes>=24.2.0`, `PyYAML>=3.11`, `jsonpatch`, `proxmoxer`, `jmespath`
- CLI binaries on the control host: `talosctl`
- Proxmox server with API access and the corresponding API key
- Proxmox storage name used in the role (defaults to `local`) must exist

## Install Requirements

1. Install talosctl: https://www.talos.dev/v1.11/talos-guides/install/talosctl/
2. Install python dependencies:
   ```bash
   python -m pip install requests kubernetes PyYAML jsonpatch proxmoxer jmespath netaddr
   ```
3. Install ansible galaxy dependencies:
   ```bash
   ansible-galaxy collection install community.general community.proxmox kubernetes.core ansible.utils
   ```
4. Install this role
   ```bash
   ansible-galaxy role install git+https://github.com/it-dfl/talos_proxmox
   ```

## Sequence

The following sequence describes what the role does in order (if called without any tags):

- Create talos configuration and patch files
- Create VMs in proxmox
- Install Talos on the VMs by applying the talos configuration; switch to trunk if vlans are enabled
- Bootstrap the talos cluster, wait for the cluster to be ready and get the kube config
- Install Multus into the cluster if enable_vlans is true
- Create all k8s resources defined in "static_resources"
- Install argocd if enabled and configure the repository and application
- Cleanup temporary files

## Minimal Example

Create a playbook calling the role:
install_cluster.yml
```yaml
- hosts: localhost
   gather_facts: true
   roles:
      - role: talos_proxmox
```

Create an inventory that overrites the default values with your needs:
inventory/<env>/group_vars/all.yml
```yaml
proxmox_server: "my-proxmox.test.local"
proxmox_api_user: "root"
proxmox_api_token_id: "my-token-id"
proxmox_api_token_secret: "my-token"

cluster_name: "k8s.test.local"
cluster_shared_ip: 192.168.0.10
mgmt_network: "192.168.0.0/24"
mgmt_gateway: "192.168.0.1"

enable_vlans: false
mgmt_vlan_id: false

k8s_vmbr: "vmbr0"

masters:
  - name: master1
    ip: 192.168.0.11
    cores: 2
    memory: 4096
    disk: 100
  - name: master2
    ip: 192.168.0.12
    cores: 2
    memory: 4096
    disk: 100
  - name: master3
    ip: 192.168.0.13
    cores: 2
    memory: 4096
    disk: 100

workers:
  - name: worker1
    ip: 192.168.0.21
    cores: 4
    memory: 8192
    disk: 100
  - name: worker2
    ip: 192.168.0.22
    cores: 4
    memory: 8192
    disk: 100
```

Now run the playbook to install the cluster:
```bash
ansible-playbook -i inventory/<env>/ install_cluster.yml
```

## Tags

You can target specific steps or do other tasks by calling the role with specific tags:
   - `configure_talos`: Create talos configuration and patch files
   - `create_vms`: Create VMs in proxmox
   - `install_talos`: Install Talos on the VMs by applying the talos configuration
   - `bootstrap_cluster`: Bootstrap the talos cluster
   - `install_multus`: Install Multus into the cluster if enable_vlans is true
   - `deploy_static_resources`: Create all k8s resources defined in "static_resources"
   - `install_argocd`: Install argocd if enabled
   - `cleanup`: Do cleanup tasks after the installation
   - `uninstall`: Uninstall the talos cluster by deleting the proxmox VMs

## How To

### Port / VLAN Setups

This section describes the different setups for the main interface of the Talos VMs in Proxmox

#### Access Port with no VLAN

```yaml
# Disable vlans to make the role assume an access port
enable_vlans: false
# Set the mgmt vlan id to false so that no vlan tag will be set on the proxmox interface
mgmt_vlan_id: false
```

### Access Port on specific VLAN

```yaml
# Disable vlans to make the role assume an access port
enable_vlans: false
# Set the mgmt vlan id to the target vlan
mgmt_vlan_id: 99
```

### Trunk Port with mgmt vlan

This is the default setup. 
This will start during the installation with an access port on the provided mgmt vlan and will switch to a trunk port during talos installation.
This will also install multus so pods can get an additional interface in a specific vlan via network attachment definitions.

```yaml
# Set enable_vlans to true for a trunk port
enable_vlans: true
# Set the mgmt vlan id to the target vlan
mgmt_vlan_id: 99
```

### Disk Encryption

Disk encryption of the primary disk is not supported by this role. However you can simply configure disk encryption of the secondary disk like this:
```yaml
workers:
  - name: worker1
    ip: 192.168.0.21
    cores: 4
    memory: 8192
    disk: 100
    storage_disk: 200 # Define a secondary disk
    storage_disk_encryption_key: "my-secret-key" # Define an encryption key
```

## Variables

The following table lists role variables (from `defaults/main.yml`), their default values, and a short description.

| Variable | Default | Description |
|---|---|---|
| `proxmox_server` | `proxmox.example.com` | Hostname or IP of the Proxmox server |
| `proxmox_port` | `8006` | Proxmox API port |
| `proxmox_url` | `https://{{ proxmox_server }}:{{ proxmox_port }}` | Base Proxmox URL (derived) |
| `proxmox_api_url` | `{{ proxmox_url }}/api2/json` | Full Proxmox API base URL (derived) |
| `proxmox_api_user` | `example_proxmox_user` | Proxmox API user for token authentication |
| `proxmox_api_token_id` | `example_token_id` | Proxmox API token id (use real token in inventory) |
| `proxmox_api_token_secret` | `example_token` | Proxmox API token secret (use real secret in inventory) |
| `proxmox_node` | `proxmox` | Proxmox cluster node to create VMs on |
| `proxmox_storage` | `local` | Proxmox storage name for ISOs/templates |
| `proxmox_vm_storage` | `local-lvm` | Proxmox storage target for VM disks |
| `proxmox_storage_url` | `{{ proxmox_api_url }}/nodes/{{ proxmox_node }}/storage/{{ proxmox_storage }}` | Proxmox storage API URL (derived) |
| `talos_version` | `v1.11.1` | Talos release version to download/install |
| `talos_qemu_agent_path` | `factory.talos.dev/image/...` | Path used to construct Talos ISO download URL |
| `talos_iso_download_url` | `https://{{ talos_qemu_agent_path }}/{{ talos_version }}/metal-amd64.iso` | Talos ISO download URL (derived) |
| `talos_installation_disk_path` | `/dev/sda` | Disk device inside VM where Talos will be installed |
| `talos_cpu_type` | `kvm64` | CPU type to pass to Proxmox VM definition |
| `cluster_name` | `k8s.example.com` | Cluster name used in Talos configs |
| `cluster_shared_ip` | `192.168.100.10` | Shared/virtual IP for control-plane services |
| `mgmt_network` | `192.168.100.0/24` | Management network CIDR for Talos machine configs |
| `mgmt_gateway` | `192.168.100.1` | Default gateway for management network |
| `enable_vlans` | `true` | Enable VLAN-aware VM/network configuration and Multus |
| `mgmt_vlan_id` | `100` | VLAN ID used for management sub-interface when VLANs enabled |
| `k8s_vmbr` | `vmbr0` | Proxmox bridge used for VM network interfaces |
| `masters` | See file | List of master VM definitions (name, ip, cores, memory, disk) |
| `workers` | See file | List of worker VM definitions (may include `storage_disk`) |
| `vlan_config` | See file | List of VLAN definitions used to create Multus NADs |
| `nameservers` | `[1.1.1.1, 8.8.8.8]` | DNS servers rendered into Talos configs |
| `timeservers` | `[time.cloudflare.com]` | NTP/time servers for Talos configs |
| `disable_ipv6` | `true` | Disable IPv6 in rendered Talos configs if true |
| `static_resources` | `[]` | List of k8s resources to apply after cluster creation (see file) |
| `install_argocd` | `false` | Whether to install ArgoCD after cluster bootstrap |
| `argocd_repo_url` | `https://github.com/example/app-of-apps.git` | Repo URL for ArgoCD app-of-apps deployment |
| `argocd_repo_type` | `git` | Type of repo for ArgoCD (git or helm) |
| `argocd_repo_revision` | `main` | Revision/branch for the ArgoCD repo |
| `argocd_app_name` | `app-of-apps` | Name of the app-of-apps application in ArgoCD |
| `argocd_app_namespace` | `argocd` | Namespace where ArgoCD will be installed |
| `argocd_app_path` | `apps` | Path inside repo for the app-of-apps manifests |
| `argocd_app_dest_namespace` | `default` | Default destination namespace used by the app-of-apps |
| `argocd_app_dest_server` | `https://kubernetes.default.svc` | Kubernetes API server address for ArgoCD apps |

## License
MIT
