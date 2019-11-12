# terraform-viptela

This repo contains terraform code for deploying the Cisco SD-WAN (Viptela) control plane components in various cloud environments.

## Requirements

- [Terraform](https://www.terraform.io).  Install with Homebrew:
    ```
    brew install terraform
    ```
- mkisofs is used to create the cloud-init ISOs.  Install with Homebrew:
    ```
    brew install cdrtools
    ```



## VMware

In the vCenter UI, create the VM templates by deploying the Viptela OVF for vManage, vEdge and vSmart.

> Note: During the deploy, in the "Select storage" section, set the virtual disk format to "Thin provisioned" to make more efficient use of the datastore disk space.

Edit the settings of each VM template and:

1. Add a "SCSI Controller" of type "LSI Logic Parallel"
1. Change "Hard disk 1" "Virtual Device Node" setting from "IDE 0" to "New SCSI controller"
1. Click OK

> Note: Do not add a second disk to the vManage template.  Terraform will do this dynamically.

Change to the vmware directory.

```
cd vmware
```

Create a `terraform.tfvars` file with the following variables set, or pass in these variables some other way (e.g. Ansible, environment variables, etc.)

```
vsphere_user      = johndoe@xyz.com
vsphere_password  = abc123
vsphere_server    = vc1.xyz.com
datacenter        = "xyz-datacenter"
cluster           = "xyz-cluster"
datastore         = "datastore1"
iso_datastore     = "datastore1"
iso_path          = "cloud-init"
vmanage_template  = "viptela-vmanage-18.4.3-genericx86-64"
vbond_template    = "viptela-edge-18.4.3-genericx86-64"
vsmart_template   = "viptela-smart-18.4.3-genericx86-64"
vedge_template    = "viptela-edge-18.4.3-genericx86-64"

vmanage_device_list = [
  {
    name = "vmanage1"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "192.168.1.2/24"
    ipv4_gateway = "192.168.1.1"
  },
  {
    name = "vmanage2"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  }
]

vsmart_device_list = [
  {
    name = "vsmart1"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  },
  {
    name = "vsmart2"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  }
]

vbond_device_list = [
  {
    name = "vbond1"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  },
  {
    name = "vbond2"
    networks = ["vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  }
]

vedge_device_list = [
  {
    name = "vedge1"
    networks = ["vmnetwork","vmnetwork","vmnetwork"]
    ipv4_address = "dhcp"
  }
]
```

> Note: the `networks` list is an ordered list of VM networks to use for each interface of the device.  For vManage/vSmart the order is eth0, eth1.  For vBond/vEdge the order is eth0, g0/0, g0/1, g0/2, g0/3.

> Note: the `*_template`, `datacenter`, `cluster`, `datastore` and `iso_datastore` values should be set to the names of the respective objects in vCenter.

> Note: `ipv4_address` is applied to VPN 0 must be set to either "dhcp" or a static IP address in address/prefix-length notation (i.e. 192.168.0.2/24).  When specifying a static IP address, `ipv4_gateway` is also required.

You can set the server and login credentials for vCenter in your environment if you do not want to put these in the `terraform.tfvars` file.  Example:

```
export TF_VAR_vsphere_user=johndoe@xyz.com
export TF_VAR_vsphere_password=abc123
export TF_VAR_vsphere_server=vc1.xyz.com
```

Run terraform.

```
$ terraform init
$ terraform plan
$ terraform apply
```

Retreive the IP addressing assigned to all control plane components.

```
$ terraform output
vbond_ip_addresses = [
  "192.168.1.209",
  "192.168.1.210",
]
vmanage_ip_addresses = [
  "192.168.1.2",
  "192.168.1.202"
]
vsmart_ip_addresses = [
  "192.168.1.211",
  "192.168.1.213",
]
vedge_ip_addresses = [
  "192.168.1.208"
]
```

Stop the VMs and delete them from vCenter.

```
$ terraform destroy
```

## AWS
Contact workshop lead to share AMI's with your AWS account.
> Note: Ability to generate AMI's from qcow image is being developed.

Deploy AWS VPC for Cisco SD-WAN controllers:
Edit Provision_VPC/my_vpc_variables.auto.tfvars.json with your region and VPC cidr_block.
> Note: CIDR block must have a prefix length less than 28 to cover subnets in 2 availability zones
```
{
    "region": "us-east-1",
    "cidr_block": "10.100.100.0/24"
}
```

With Provision_VPC as your current working directory, run terraform.
```
$ terraform init
$ terraform plan
$ terraform apply
```

Deploy Controllers into VPC:
Edit Provision_Instances/my_instances_variables.auto.tfvars.json with appropriate settings.
```
{
    "vbond_instances_type": "c5.large",
    "vsmart_instances_type": "c5.xlarge",
    "vmanage_instances_type": "c5.4xlarge",
    "vbond_ami": "ami-085c4adc58506ad83",
    "vmanage_ami": "ami-06850b5d3d92800e7",
    "vsmart_ami": "ami-0079a97de83928496",
    "vbond_count": "1",
    "vmanage_count": "1",
    "vsmart_count": "1"
}
```

With Provision Instances as your current working directory, run terraform
```
$ terraform init
$ terraform plan
$ terraform apply
```

Retreive the IP addressing assigned to all control plane components.
```
$ terraform output
vbonds_vbondEth0EIP = [
  "3.231.238.177",
]
vbonds_vbondEth0Ip = [
  "10.100.100.80",
]
vbonds_vbondEth1EIP = [
  "3.231.90.13",
]
vbonds_vbondEth1Ip = [
  [
    "10.100.100.7",
  ],
]
vmanages_vmanageEth0EIP = [
  "3.232.23.107",
]
vmanages_vmanageEth0Ip = [
  "10.100.100.67",
]
vmanages_vmanageEth1EIP = [
  "3.230.210.217",
]
vmanages_vmanageEth1Ip = [
  [
    "10.100.100.59",
  ],
]
vsmarts_vsmartEth0EIP = [
  "3.230.217.130",
  "34.193.188.60",
]
vsmarts_vsmartEth0Ip = [
  "10.100.100.52",
  "10.100.100.212",
]
vsmarts_vsmartEth1EIP = [
  "3.232.82.69",
  "3.212.251.219",
]
vsmarts_vsmartEth1Ip = [
  [
    "10.100.100.85",
  ],
  [
    "10.100.100.134",
  ],
]
```

To terminate instances, go to the Provision_Instances directory and run:
```
$ terraform destroy -force
```

To destroy the empty controllers' VPC, go to the Provision_VPC directory and run:
```
$ terraform destroy -force
```

##Azure
###Coming soon...

You can set your ARM credentials in your environment.  See below:
```
export TF_VAR_ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export TF_VAR_ARM_CLIENT_SECRET="00000000-0000-0000-0000-000000000000"
export TF_VAR_ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
export TF_VAR_ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
```
