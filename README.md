#  Deploying a vSphere VM with Terraform and Injecting SSH Key via Cloud-Init

This guide shows how to use Terraform to provision a virtual machine on VMware vSphere (vCenter) using a cloud-init enabled template. The VM will have:
- A **custom hostname** (`HamedVM`)
- A **new user** (`ansible`)
- The **Ansible controller's public SSH key** pre-installed

---

##  Requirements

- Access to vCenter (e.g. `192.168.55.11`)  
- A VM template with **cloud-init installed** (e.g. `ubuntu-24.04-template`)  
- Ansible controller's **public SSH key** (e.g. from `~/.ssh/id_rsa.pub`)  
- Terraform v1.x  
- Terraform VMware vSphere Provider  
- vCenter credentials (e.g. `administrator@vsphere.local`)  

---

##  Project File Structure

Create a working directory (e.g. `~/vmware`) with the following files:

```
vmware/
├── main.tf
├── variables.tf
├── secrets.tfvars
├── cloud-init.yaml
```

---

##  Step 1: Save the Ansible Public SSH Key

Open the Ansible controller and copy the public SSH key:

```bash
cat ~/.ssh/id_rsa.pub
```

Paste the output into your `cloud-init.yaml` file in the next step.

---

##  Step 2: Create the `cloud-init.yaml` File

```yaml
#cloud-config

# Set the hostname of the new virtual machine
hostname: HamedVM

# Create a new user named 'ansible' with sudo privileges
users:
  - name: ansible
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash

    # Add the public SSH key of the Ansible controller
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD...your_ansible_public_key_here
```

>  Replace the SSH key above with your real public key.

---

##  Step 3: Create `main.tf`

```hcl
terraform {
  required_providers {
    vsphere = {
      source  = "vmware/vsphere"
      version = "~> 2.5"
    }
  }
}

provider "vsphere" {
  user                 = "administrator@vsphere.local"
  password             = var.vsphere_password
  vsphere_server       = "192.168.55.11"
  allow_unverified_ssl = true
}

data "vsphere_datacenter" "dc" {
  name = "Sabzbahar Datacenter"
}

data "vsphere_compute_cluster" "cluster" {
  name          = "Sabz Cluster"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_datastore" "datastore" {
  name          = "datastore1"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_network" "network" {
  name          = "VM Network"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_virtual_machine" "template" {
  name          = "ubuntu-24.04-template"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "template_file" "cloudinit" {
  template = file("${path.module}/cloud-init.yaml")
}

resource "vsphere_virtual_machine" "vm" {
  name             = "HamedVM"
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.datastore.id

  num_cpus = 2
  memory   = 2048
  guest_id = data.vsphere_virtual_machine.template.guest_id
  scsi_type = data.vsphere_virtual_machine.template.scsi_type

  network_interface {
    network_id   = data.vsphere_network.network.id
    adapter_type = data.vsphere_virtual_machine.template.network_interface_types[0]
  }

  disk {
    label            = "disk0"
    size             = 25
    thin_provisioned = data.vsphere_virtual_machine.template.disks[0].thin_provisioned
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.template.id

    customize {
      linux_options {
        host_name = "HamedVM"
        domain    = "local"
      }

      network_interface {
        ipv4_address = "192.168.55.50"
        ipv4_netmask = 24
      }

      ipv4_gateway    = "192.168.55.1"
      dns_server_list = ["192.168.55.2"]
    }

    ovf_environment {
      properties = {
        "user-data" = base64encode(data.template_file.cloudinit.rendered)
      }
    }
  }
}
```

---

##  Step 4: `variables.tf`

```hcl
variable "vsphere_password" {
  description = "Password for vSphere user"
  type        = string
  sensitive   = true
}
```

---

##  Step 5: `secrets.tfvars`

```hcl
vsphere_password = "YourSecurePassword"
```

---

##  Step 6: Run Terraform

```bash
terraform init
terraform plan -var-file="secrets.tfvars"
terraform apply -var-file="secrets.tfvars"
```

Type `yes` to confirm.

---

##  Result

- A VM named `HamedVM` will be created
- User `ansible` will be present inside the VM
- SSH key will be pre-installed
- You can access it with:

```bash
ssh ansible@192.168.55.50
```

---
