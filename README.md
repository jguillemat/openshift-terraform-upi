## OpenShift terraform UPI

The follow procedure assume you have installed terraform and you have an **install-config.yaml**

The repository contain a bash script **install.sh** to generate the ignition files required by the **openshift-install**, or you can generate by your own and copy to the web server **/var/www/html**. Terraform expect read from the directory mentioned.

### Requirements
- Understanding of the syntax of the **openshift-install.yaml**
- install-config.yaml with the OpenShift configurations and vcenter configurations
- Web server (httpd) to serve the ignition files
- terraform
- Internet connection 

#### Modify the template of the **install-config.yaml**
Yo can use the **intall-config.yaml** and change any configuration required

#### Adapt the bash script changing the values of your pullsecret and SSH key
```bash
#!/bin/bash
set -xe
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

rm -fr deployment
mkdir deployment

cp install-config.yaml deployment/

PULL_SECRET=$(cat ../.pull-secret | tr -d '\n\r\t ')
sed -i "s/PULL_SECRET/${PULL_SECRET}/" deployment/install-config.yaml

SSH_KEY=$(cat ~/.ssh/id_rsa.pub | tr -d '\n\r\t' | sed -r 's/\//\\\//g')
sed -i "s/SSH_KEY/${SSH_KEY}/" deployment/install-config.yaml

openshift-install create manifests --dir=deployment

sed -i 's/true/false/g' deployment/manifests/cluster-scheduler-02-config.yml
rm -f deployment/openshift/99_openshift-cluster-api_master-machines-*.yaml deployment/openshift/99_openshift-cluster-api_worker-machineset-*.yaml

openshift-install create ignition-configs --dir=deployment

sudo cp -f deployment/*.ign /var/www/html/

sudo chmod 755 /var/www/html/*
sudo chcon -R -t httpd_sys_content_t /var/www/html
sudo restorecon -R -v /var/www/html

export KUBECONFIG="${DIR}/deployment/auth/kubeconfig"
```

## Terraform 
Change to the folder terraform.

The folder **vars** contain the file **common.tfvars**, this file you can provide the vcenter configuration, and the nodes values as cpu and memory, also the networking values. The variable file contain a dictionary with nested values of your VMs, you can add many workers, infra and other kind of nodes. The **ip** list (array) must be the same length as the **hostname** value. The example shopws 3 worker, but can be 2 or 4 or more. The same infra. The example shows the **sotorage.ip** and **storage.hostname** empty, that makes when you run the **plan** in one step, the storage VMs skip the loop.  
```bash
## NODE INFORMATIOM
node_configs = {
  bootstrap = {
    ip       = "192.168.0.30"
    hostname = "bootstrap"
    cpu      = 4
    memory   = 16384
    disk     = 120
  }
  master = {
    ip       = ["192.168.0.31", "192.168.0.32", "192.168.0.33"]
    hostname = ["master01", "master02", "master03"]
    cpu      = 4
    memory   = 16384
    disk     = 120
  }
  worker = {
    ip       = ["192.168.0.34", "192.168.0.35", "192.168.0.36"]
    hostname = ["worker01", "worker02", "worker03"]
    cpu      = 8
    memory   = 16384
    disk     = 120
  }
  infra = {
    ip       = ["192.168.0.37", "192.168.0.38", "192.168.0.39"]
    hostname = ["infra01", "infra02", "infra03"]
    cpu      = 4
    memory   = 16384
    disk     = 120
  }
  storage = {
    ip       = []
    hostname = []
    cpu      = 4
    memory   = 16384
    disk     = 120
  }
}
```

### Deploy the VMs
The terraform folder contain 3 bash scripts: plan, apply, destroy. You can use to deploy the infrastructure in one step.
```bash 
$ ./plan 
```
Will execute the plan you can visualize the resources to be created.

```bash 
$ ./apply 
```
Will deploy the cluster

```bash 
$ ./destroy
```
Will delete all the VM, and folders created by terraform. It won't destroy none of the volumes, CSI volumes created by the cluster. just the original VMs, disks and folders

To destroy the bootstrap once the process is finish see the section [Destroy the bootstrap node after the API is finish](#Destroy-the-bootstrap-node-after-the-API-is-finish)

#### Deploy in many steps
If you want to deploy partially the cluster you can add the **-target** flag with **plan** and **destroy**
```bash 
$ ./plan -target=module.bootstrap
Initializing modules...

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/template from the dependency lock file
.
.
.
Plan: 2 to add, 0 to change, 0 to destroy.
╷
│ Warning: Resource targeting is in effect
│
````
Will create only the bootstrap node

```bash 
$ ./plan -target=module.master
````
Will create only the master nodes

```bash 
$ ./plan -target=module.worker
````
Will create only the worker nodes

```bash 
$ ./plan -target=module.infra
````
Will create only the infra nodes

```bash 
$ ./plan -target=module.storage
````
Will create only the storage nodes

After run the plan with the tag is just needed to **apply** to deploy the **-target** plan
```bash 
$ ./apply
Initializing modules...

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/vsphere from the dependency lock file
- Reusing previous version of hashicorp/template from the dependency lock file
- Using previously-installed hashicorp/vsphere v2.2.0
- Using previously-installed hashicorp/template v2.2.0
.
.
.
vsphere_folder.cluster: Creating...
vsphere_folder.cluster: Creation complete after 1s [id=group-v4044]
.
.
.
module.bootstrap.vsphere_virtual_machine.openshift: Still creating... [40s elapsed]
│ Warning: Applied changes may be incomplete
│
│ The plan was created with the -target option in effect, so some changes requested in the
│ configuration may have been ignored and the output values may not be fully updated. Run the
│ following command to verify that no other changes are pending:
│     terraform plan
│
│ Note that the -target option is not suitable for routine use, and is provided only for exceptional
│ situations such as recovering from errors or mistakes, or when Terraform specifically suggests to
│ use it as part of an error message.
╵

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
````

### Destroy the bootstrap node after the API is finish
Use **destroy** script with the **-target=module.bootstrap** to destroy the bootstrap node when the **openshift-install** show is safe to delete

```bash
./destroy -target=module.bootstrap
data.vsphere_datacenter.dc: Reading...
data.vsphere_datacenter.dc: Read complete after 0s [id=datacenter-3]
.
.
.
Plan: 0 to add, 0 to change, 1 to destroy.
╷
│ Warning: Resource targeting is in effect
│
│ You are creating a plan with the -target option, which means that the result of this plan may not
│ represent all of the changes requested by the current configuration.
│
│ The -target option is not for routine use, and is provided only for exceptional situations such as
│ recovering from errors or mistakes, or when Terraform specifically suggests to use it as part of an
│ error message.
╵

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: 
```
Type **yes** and the bootstrap will be deleted




### Terraform structure 
```bash 
.
├── apply
├── destroy
├── main.tf
├── modules
│   ├── bootstrap
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   ├── templates
│   │   │   └── merge-bootstrap.tpl
│   │   └── vars.tf
│   ├── nodes
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── vars.tf
│   ├── nodes-master
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── vars.tf
│   └── nodes-storage
│       ├── main.tf
│       ├── outputs.tf
│       └── vars.tf
├── plan
└── vars
    └── common.tfvars
```

