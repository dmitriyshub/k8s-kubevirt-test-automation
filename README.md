## Ansible Automation for Kubevirt on Kubernetes Full Functionality Tests with External Access by MetalLB

---

#### This Ansible Role Tests the VM functionality for an Openshift\Kubernetes Cluster post-upgrade. It initializes several objects in the cluster, aiming to create a fully functional virtual machines servers and tests the functionality 

To run the Ansible role without modifications, ensure that your infrastructure includes:

1. OpenShift/Kubernetes Cluster 
  - You need a functioning OpenShift or Kubernetes cluster with 2 Workers (min)

2. CNV/KubeVirt Operator for Managing Virtual Machines
  - OpenShift CNV (OpenShift Virtualization) or KubeVirt must be installed, allows you to run and manage VMs alongside container workloads within your cluster
3. MetalLB with a Configured Available IP Address Pool
  - MetalLB is a load-balancer implementation for bare-metal Kubernetes clusters. It allows you to assign external IP addresses to services within the cluster
4. VLAN network bridge
  - You need to set up a VLAN network bridge.
5. OpenShift VM templates and Storage Classes with 3 Availability Zones
  - 3 Different VM tempaltes for each OS Version (rhel7-9-az-a etc.)
  - 3 Different Storage Classes for each Availability Zones (storageClassName az-a etc.)

Setting up this environment will provide the necessary resources and configurations required by the Ansible role to function correctly otherwise please change the role functinality 

---
### Role Test Flow Actions
---

- **Initialization**: 
  - Generate `ssh` key pair for the `VMs`
  - Create an Openshift `Project`
  - Set up `ConfigMap` with `index.html` file content
  - Create a `Secret` with the `ssh` public key

- **Network Configuration**:
  - Implement `NetworkAttachmentDefinition` with a bridge
  - Establish a `Service` type `LoadBalancer` with open ports `22` and `80`
  - Configure a `yum` repository and install the `httpd` service
  - Install and configure the `firewalld` service

- **Storage Management**:
  - Create, mount, and manage new `PVCs` and `lvm` for the virtual machine
  - File management on `lvm`, including md5sum checksum storage and comparison
  - Extend `PVCs` and resize `lvm` and compare the checksums

- **VM Operations**:
  - Execute VM `Live Migration`, `Cloning`, and connectivity checks
  - Handle `VM Snapshot`, `VM Restore`, and subsequent checks

- **VLAN Bridge Network Configuration**:
  - Assign Static `IP` for Bridge `NIC` Device
  - Ensure Bridge `NIC` Network Connectivity between the `VMs`

- **Resource Managment**:
  - Check and Compare the `virt-launcher` Pod and `VMI` CPU limits to `VM` limits

- **Cleanup**:
  - Delete all objects, retaining the project

#### Additional Tasks:
  - Create VMs from OpenShift templates and verify their `Ready` status

---
### Tools and Versions
---

| Tool                       | Description                   |  Tested on Version            | Optional |
|----------------------------|-------------------------------|-------------------------------|----------|
| `Ansible`                  | Ansible Control Node Client   | `2.15.1 ` & `2.9.27`          |    No    |
| `Jinja`                    | Jinja Template Engine         | `3.1.2` & `2.10.1`            |    No    |
| `Python`                   | Python Source and Destination | `3.11.4` & `3.6.8` -> `2.7.5` |    No    |
| `Kubernetes.core`          | Ansible K8s Module            | `2.4.0`                       |    No    |
| `community.kubernetes`     | Old Ansible K8s Module        |                               |    Yes   |
| `OpenShift`                | Openshift Cluster             | `4.10.16` & `4.12.22`         |    Yes   |
| `Openshift Virtualization` | Kubevirt Operator             | `4.10.1` & `4.11.4`           |    No    |
| `MetalLB`                  | MetalLB Operator              | `4.10.0` & `4.12.0`           |    No    |
| `OpenShift Data Foundation`| ODF Operator                  | `4.10.13` & `4.12.4`          |    Yes   |


---
### Project Tree Structure
---

- Refer to the following tree structure
```bash
$PROJECT/
|-- files/ # Directory for the j2 templates output YAML files
    |-- *.yml
|-- roles/
----------------------------------------
    |-- vm-user-stories/
        |-- defaults/ # Role defaults
            |-- main.yml
        |-- files/
            |-- index.html
        |-- handlers/
            |-- main.yml
        |-- tasks/
            |-- main.yml
            |-- *.yml
        |-- templates/
            |-- *.j2
        |-- vars/ # Role vars
            |-- main.yml
----------------------------------------
    |-- other_role/ # OPTIONAL
|-- secret/ # Contains authentication token and ca.crt
    |-- token
    |-- ca.crt
|-- ansible.cfg 
|-- inventory.ini
|-- main.yml # Main playbook
|-- pre_main.yml # Optional playbook for ca bundle setup
```

---
### Prerequisites and Usage
---
Use the project tree structure for reference

1. Clone this role to the `roles` directory in your project.
```bash
git clone git@github.com:dmitriyshub/k8s-kubevirt-test-auto.git $HOME/$PROJECT
```
2. Create `inventory.ini` and `ansible.cfg` files at the project level
```ini
# inventory.ini file content
localhost ansible_connection=local

# ansible.cfg file content
[defaults]
inventory = ./inventory.ini
callbacks_enabled = profile_tasks
deprecation_warnings=False
```

3. Prepare the `files` and `secret` directories in your project
```bash
mkdir $PROJECT/{files,secret}
```
4. Make sure that you have  a `token` and `ca.crt` bundle available at the specified paths
```bash
# default token path
$PROJECT/secret/token
# default ca.crt path
$PROJECT/secret/ca.crt
```

5. Optionally, create the `pre_main.yml` playbook to save the `ca.crt` bundle
```yaml
- name: Create local cluster CA bundle file
  hosts: localhost
  become: false
  gather_facts: false
  module_defaults:
    group/k8s:
      host: "{{ API_URL }}"
      api_key: "{{ AUTH_TOKEN }}"
  tasks:
    - name: Create CA bundle 
      include_role:
        name: vm-user-stories
        tasks_from: prerequisites_ca.yml
```

6. Create the `main.yml` playbook for the tests
```yaml
---
- name: VM User Stories Testing OCP cluster
  hosts: localhost
  become: false
  gather_facts: false
  module_defaults:
    group/k8s:
      host: "{{ API_URL }}"
      api_key: "{{ AUTH_TOKEN }}"
      ca_cert: "{{ CA_CERT }}"
  roles:
    # rhel7.9 az-a os image test
    - name: "rhel7.9 test az-a"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel7-9-az-a
        VM_NAME: rhel7-test-vm-az-a
        YUM_REPO_URL: http://yum.com/rhel-7/rhel-7-server-rpms # Private yum repository url
        STORAGE_CLASS_NAME: az-a
    # rhel7.9 az-b os image test
    - name: "rhel7.9 test az-b"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel7-9-az-b
        VM_NAME: rhel7-test-vm-az-b 
        YUM_REPO_URL: http://yum.com/rhel-7/rhel-7-server-rpms
        STORAGE_CLASS_NAME: az-b
    # rhel7.9 az-c os image test
    - name: "rhel7.9 test az-c"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel7-9-az-c 
        VM_NAME: rhel7-test-vm-az-c
        YUM_REPO_URL: http://yum.com/rhel-7/rhel-7-server-rpms
        STORAGE_CLASS_NAME: az-c
        affinity: false
    # rhel8.4 az-a os image test
    - name: "rhel8.4 test az-a"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel8-4-az-a
        VM_NAME: rhel8-test-vm-az-a
        YUM_REPO_URL: http://yum.com/rhel-8/rhel-8-for-x86_64-baseos-rpms 
        STORAGE_CLASS_NAME: az-a
    # rhel8.4 az-b os image test
    - name: "rhel8.4 test az-b"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel8-4-az-b
        VM_NAME: rhel8-test-vm-az-b
        YUM_REPO_URL: http://yum.com/rhel-8/rhel-8-for-x86_64-baseos-rpms
        STORAGE_CLASS_NAME: az-b
    # rhel8.4 az-c os image test
    - name: "rhel8.4 test az-c"
      role: vm-user-stories
      vars:
        OS_PVC_NAME: rhel8-4-az-c
        VM_NAME: rhel8-test-vm-az-c
        YUM_REPO_URL: http://yum.com/rhel-8/rhel-8-for-x86_64-baseos-rpms
        STORAGE_CLASS_NAME: az-c
        affinity: false
  tasks:
    # rhel7.9 multi az templates test
    - name: rhel7 templates 
      include_role:
        name: vm-user-stories
        tasks_from: vms_from_templates
      vars:
        TEMPLATE_NAME:
          temp1: rhel7-9-az-a
          temp2: rhel7-9-az-b
          temp3: rhel7-9-az-c
    # rhel8.4 multi az templates test
    - name: rhel8 templates 
      include_role:
        name: vm-user-stories
        tasks_from: vms_from_templates
      vars:
        TEMPLATE_NAME:
          temp1: rhel8-4-az-a
          temp2: rhel8-4-az-b
          temp3: rhel8-4-az-c
    # win 2016 multi az templates test
    - name: win16 templates 
      include_role:
        name: vm-user-stories
        tasks_from: vms_from_templates
      vars:
        TEMPLATE_NAME:
          temp7: windows-2016-az-a
          temp8: windows-2016-az-b
          temp9: windows-2016-az-c
```

7. Execute the following command to run the playbook 
```shell
pip3 install --upgrade --user openshift
oc login --token=<your-token> --server=<your-cluster-api> # or export kubeconfig
ansible-playbook pre-main.yml # Optional
ansible-playbook main.yml
```

8. Use the generated SSH key pair for debugging
```bash
# default ssh path
$HOME/.ssh/vm-key
$HOME/.ssh/vm-key.pub
```

---
### Setting Up The Environment 
---

1. Modify the cluster `API_URL` variable and validate all role variables (e.g., `IP_ADDRESS_POOL`, `BRIDGE_NAME`,`VLAN_NUM` `YUM_REPO_URL`)

```yaml
API_URL: https://api.cluster.domain:6443
IP_ADDRESS_POOL: <metallb-address-pool-name>
BRIDGE_NAME: <node-linux-bridge-name>
VLAN_NUM: <node-linux-bridge-vlan-number>
YUM_REPO_URL: <yum-server-url>
```

2. Ensure the following:
   - Availability of `OS_PVC_PROJECT_NAME` with `OS_PVC` with OS images and the `STORAGE_CLASS_NAME` with provisioner
   
   - MetalLB `IP_ADDRESS_POOL` is set up with `autoAssign: true`

   - Correct configuration for `BRIDGE_NAME` and `VLAN_NUM` (if `dhcp` is available you can change `vlan` variable to `dhcp`)
   ```
   # cnv-bridge bridge-net with vlan available on your cluster nodes
   {"type":"cnv-bridge","bridge":"bridge-net","vlan":32}
   ```

   - Correct `VM_BRIDGE_IP`,`CLONE_BRIDGE_IP` and `BRIDGE_GATEWAY` related to the vlan configuration

   - Accessible `YUM_REPO_URL` server with packages

   - To perform LiveMigration change `livemigration` to `true`

   - To delete affinity and antiaffinity rules change `affinity` to `false`

   - To make the lvm tasks more idempotent change `ansible_ver` to `2.15` (Ensure that your ansible client is updated)

---
### Role Variables
---

Here is a list of primary variables used in this role:

| Variable                | Default Value                                                  | Description                   | Source        |
|-------------------------|----------------------------------------------------------------|-------------------------------|---------------|
| `API_URL`               | `https://api.cluster-0.com:6443`                               | Cluster API endpoint URL      | role vars     |
| `AUTH_TOKEN`            | `lookup('file','secret/token')`                                | Cluster authentication token  | role vars     |
| `CA_CERT`               | `secret/ca.crt`                                                | Cluster CA bundle certificate | role vars     |
| `HOME_DIR`              | `lookup('env','HOME')`                                         | Local home directory path     | role vars     |
| `SSH_PUBLIC_KEY`        | `{{ HOME_DIR }}/.ssh/vm-key.pub`                               | Local public ssh key path     | role vars     |
| `SSH_PRIVATE_KEY`       | `{{ HOME_DIR }}/.ssh/vm-key`                                   | Local private ssh key path    | role vars     |
| `PROJECT`               | `post-upgrade-testing`                                         | Project name for the objects  | role defaults |
| `VM_NAME`               | `post-upgrade-testing-vm`                                      | Main names for all objects    | role defaults |
| `OS_PVC_NAME`           | `rhel7-9-az-a`                                                 | Operating System PVC name     | role defaults |
| `OS_PVC_PROJECT_NAME`   | `openshift-virtualization-os-images`                           | Operating System PVC namespace| role defaults |
| `STORAGE_CLASS_NAME`    | `az-a`                                                         | Storage Class for PVCs        | role defaults |
| `IP_ADDRESS_POOL`       | `default`                                                      | MetalLB Address pool name     | role defaults |
| `YUM_REPO_URL`          | `http://yum.com/rhel-7/rhel-7-server-rpms`                     | Yum server URL                | role defaults |
| `BRIDGE_NAME`           | `bridge-net`                                                   | Bridge network name           | role defaults |
| `VLAN_NUM`              | `32`                                                           | Bridge network number         | role defaults |
| `TEMPLATE_NAME`         | `temp1: rhel7-9-az-a,temp2: rhel7-9-az-b,temp3: rhel7-9-az-c`..| Bridge network name           | role defaults |
| `VM_USER`               | `cloud-user`                                                   | Virtual Machine username      | role defaults |
| `VM_PASSWORD`           | `test`                                                         | Virtual Machine password      | role defaults |
| `CONFIG_SERIAL`         | `config150759a777`                                             | ConfigMap Disk serial number  | role defaults |
| `PVC1_SERIAL`           | `pvc111150759a777`                                             | PVC1 LVM Disk serial number   | role defaults |
| `PVC2_SERIAL`           | `pvc222150759a777`                                             | PVC2 LVM Disk serial number   | role defaults |
| `CM_MOUNT_PATH`         | `/mnt/configmap`                                               | ConfigMap Disk mount point    | role defaults |
| `LVM_MOUNT_PATH`        | `/mnt/lvm_mount_point`                                         | LVM Disk mount point          | role defaults |
| `condition`             | `none` (`none`/`new_pvc`/`clone`)                              | Default condition for j2      | role defaults |
| `vlan`                  | `static` (`static`/`dhcp`)                                     | vlan type( static/dynamic ip) | role defaults |
| `ansible_ver`           | `2.9` (`2.9`/`2.15`)                                           | Ansible version               | role defaults |
| `livemigration`         | `false`                                                        | live migration task           | role defaults |
| `affinity`              | `true`                                                         | affinity and antiaffinity rules| role defaults|
| `VM_BRIDGE_IP`          | `192.168.32.199`                                               | Bridge NIC IP address         | role defaults |
| `CLONE_BRIDGE_IP`       | `192.168.32.198`                                               | Bridge NIC IP address (Clone) | role defaults |
| `BRIDGE_GATEWAY`        | `192.168.32.254`                                               | Bridge NIC Gateway IP address | role defaults |
| `VM_BRIDGE_MAC`         | `52:54:00:**:**:**` ( `**` randomly generated)                 | Bridge NIC Mac address        | runtime       |
| `CLONE_BRIDGE_MAC`      | `52:54:00:**:**:**` ( `**` randomly generated)                 | Bridge NIC Mac address (Clone)| runtime       |
| `ssh_key`               | `lookup('file', SSH_PUBLIC_KEY )`                              | Store the ssh pub key         | runtime       |
| `devices_by_serial`     | `devices_by_serial \| default([]) + [item.key]`                | Store disk device names       | runtime       |
| `file_md5sum`           | `file_stat.stat.checksum`                                      | Store file md5sum checksum    | runtime       |
| `new_file_md5sum`       | `new_file_stat.stat.checksum`                                  | Store new file md5sum checksum| runtime       |
| `service_ip`            | `servicelb_info.result.status.loadBalancer.ingress[0].ip`      | Store Service lb ip           | runtime       |
| `clone_service_ip`      | `servicelb_info.result.status.loadBalancer.ingress[0].ip`      | Store Service lb ip (Clone)   | runtime       |
| `host_name`             | `route_info.result.spec.host`                                  | Store Route host name         | runtime       |
| `clone_host_name`       | `route_info.result.spec.host`                                  | Store Route host name (Clone) | runtime       |
| `index_html_content`    | `index_html['content'] \| b64decode`                           | Store index.html content      | runtime       |
| `bridge_ip_address`     | `remote_facts['ansible_facts'][item]['ipv4']['address']`       | Bridge NIC IP address         | runtime       |
| `clone_bridge_ip_address`| `clone_remote_facts['ansible_facts'][item]['ipv4']['address']`| Bridge NIC IP address(Clone)  | runtime       |
| `bridge_nic_name`       | `remote_facts['ansible_facts'][item]['device']`                | Bridge NIC name               | runtime       |
| `clone_bridge_nic_name` | `clone_remote_facts['ansible_facts'][item]['device']`          | Bridge NIC name (Clone)       | runtime       |
| `vm_limits`             | `vm_info.resources[0].spec.template.spec.domain.resources.limits.cpu` | vm cpu limits          | runtime       |
| `vmi_limits`            | `vmi_info.resources[0].spec.domain.resources.limits.cpu`       | vmi cpu limits                | runtime       |
| `pod_limits`            | `pod_info.resources[0].spec.containers[0].resources.limits.cpu`| virt-launcher pod cpu limits  | runtime       |

---
### Full Playbook Outputs
---

```bash
PLAY RECAP *******************************************************************************************************
# Last Play from the bastion with templates
localhost                  : ok=1056 changed=323  unreachable=0    failed=0    skipped=90   rescued=0    ignored=0   
# Full Last Play from local machine
localhost                  : ok=1029 changed=320  unreachable=0    failed=0    skipped=66   rescued=0    ignored=0 
Sunday 10 September 2023  01:12:36 +0300 (0:00:02.593)       0:58:39.431 ****** 
```
---