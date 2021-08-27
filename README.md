ocp4-install
=================

This role takes care of installing Red Hat OpenShift 4.x platform.

Requirements
------------

- Host needs to have the following components installed before trying to deploy OpenShift.

ansible-role-cornerstone
https://github.com/kenhitchcock/ansible-role-cornerstone

Azure Components
Download and install the requirements files for Azure
https://github.com/ansible-collections/azure/blob/dev/requirements-azure.txt
sudo pip install -r requirements-azure.txt
ansible-galaxy collection install azure.azcollection


Role Variables
--------------

```
# =========================================================================== #
# Common variables for all platforms                                          #
# =========================================================================== #

# Name of the OpenShift cluster
ocp4_install_cluster_name: testcluster

# Pull secret downloaded from the Red Hat cluster portal. Must contain all the 
# content between the '' in the pull secret file.
ocp4_install_pullSecret: 'ADD PULL SECRET HERE'

# ssh pub key that will be used to ssh into nodes.
ocp4_install_sshKey: "ADD PUB KEY HERE"

# Node count
ocp4_install_master_replicas: 3
ocp4_install_worker_replicas: 3

# Network config
ocp4_install_clusterNetwork_cidr: 10.128.0.0/14
ocp4_install_clusterNetwork_hostPrefix: 23
ocp4_install_machineNetwork_cidr: 10.0.0.0/16
ocp4_install_machineNetwork_networkType: OpenShiftSDN
ocp4_install_serviceNetwork_cidr: 172.30.0.0/16


# =========================================================================== #
# Azure Variables                                                             #
# =========================================================================== #

# ocp4_install_baseDomain is the dns domain used for the cluster. 
# It will be added to Azure if not present, HOWEVER it will not update the
# registar with the Azure dns servers. You will have to do that manually
# You can do it during the deployment but risk the OpenShift install failing
# if dns is not updated quick enough 

ocp4_install_baseDomain: openshiftlab.ml

# Resource group that the base domain is configured in.
ocp4_install_baseDomain_resourcegroup: cs-rg

# This resource group will be created if it does not exist. It is the resource  
# group OpenShift will be installed in.
ocp4_install_resourcegroup: cs-rg-testcluster

# Azure Region
ocp4_install_location: uksouth

# Azure vm sizing. Check the Azure documentation for different size options.
ocp4_install_azure_it_master_diskSizeGB: 1024
ocp4_install_azure_it_master_instance_type: Standard_D8s_v3
ocp4_install_azure_it_worker_instance_type: Standard_D2s_v3
ocp4_install_azure_it_worker_diskSizeGB: 512

# Azure related configuration. You can add more but be sure to stick to the
# name: and value: parameters.
# - baseDomainResourceGroupName is the resource group used for the domain name for the cluster
# - region is the Azure region
# - outboundType is how you want the cluster accessed from the outside world.
# - cloudName is what Azure calls themselves or something like that. Check the offcial 
#   documentation for more info.

install_azure_it_worker_config_az_config:
  - name: baseDomainResourceGroupName
    value: "cs-rg"
  - name: region
    value: "uksouth"
  - name: resourceGroupName
    value: "cs-rg-testcluster"
  - name: outboundType
    value: "Loadbalancer"
  - name: cloudName
    value: "AzurePublicCloud"

# An Availability Zone is a high-availability offering that protects your applications and data from datacenter failures. Availability Zones are unique physical locations within an Azure region. Choose if you want 1 or 2 or 3 zones when deploying your cluster.
ocp4_install_azure_it_worker_config_zones:
  - 1
  - 2
  - 3

# Azure subscriptionid and tenantid can be found with the azure-cli by running the command.
# az account show
ocp4_install_azure_subscriptionid: <REPLACE ME>
ocp4_install_azure_tenantid: <REPLACE ME>

# Azure clientid and client secret are from the principal account you need to create 
# as a prerequisite for OpenShift 4 install on Azure. 
# If you have not already created the account the command is below.
# az ad sp create-for-rbac --role Contributor --name <service_principal>
ocp4_install_azure_clientid: <REPLACE ME>
ocp4_install_azure_clientsecret: <REPLACE ME>

```

Example Playbook
----------------

```
- name: "Build Instance"  
  hosts: localhost
  collections:
    - azure.azcollection
  become: true
  vars_files:
    - ../vars/azure_basicvm.yml

  tasks:
    - include_role:
        name: ../roles/ansible-role-cornerstone

- name: "Wait for VM to become live."
  hosts: bastion
  become: true
  gather_facts: false
  vars_files:
    - ../vars/azure_basicvm.yml

  tasks:
    - name: "Wait for system to become reachable over ssh, wait 2 minutes before timeout."
      wait_for_connection:
        timeout: 30
      when: vm_state == "present"

- name: "Deploy OpenShift."
  hosts: bastion
  become: true
  gather_facts: true
  vars_files:
    - ../vars/azure_basicvm.yml

  tasks:
    - include_role:
        name: ../roles/ansible-role-ocp4-install

```

Example Inventory
----------------

```
cornerstone_prefix: cs
cornerstone_platform: azure
cornerstone_ssh_user: azureuser
cornerstone_ssh_key_path: /home/user/.ssh/id_rsa

cornerstone_ssh_admin_username: azureadmin
cornerstone_ssh_admin_pubkey: <ssh pub key>

vm_state: present

guests:
  bastion:
    cornerstone_platform: azure
    cornerstone_azure_vm_disk_managed: Standard_LRS
    cornerstone_working_dir: '/tmp/'
    cornerstone_vm_state: "{{vm_state}}"
    cornerstone_vm_name: "bastion"
    cornerstone_storage_type: StandardSSD_LRS
    cornerstone_location: uksouth
    cornerstone_azure_resource_group: "{{ cornerstone_prefix }}-rg"
    cornerstone_virtual_network_name: "{{ cornerstone_prefix }}-vnet"
    cornerstone_subnet_name: "{{ cornerstone_prefix }}subnet"
    cornerstone_azure_subnet_cidr: "10.1.0.0/20"
    cornerstone_vm_assign_public_ip: false
    cornerstone_vm_public_ip:
    cornerstone_public_private_ip: public
    cornerstone_publicip_allocation_method: Dynamic
    cornerstone_publicip_domain_name: null
    cornerstone_vm_flavour: Standard_D2s_v3
    cornerstone_vm_azure_image: rhel8-gm
    cornerstone_vm_image_offer: "RHEL"
    cornerstone_vm_image_publisher: "RedHat"
    cornerstone_vm_image_sku: "8.1"
    cornerstone_vm_image_version: "latest"
    cornerstone_vm_os_disk_size: 32
    cornerstone_vm_data_disk: false
    cornerstone_tag_purpose: "bastion"
    cornerstone_tag_role: "testing"


ocp4_install_baseDomain: openshiftlab.ml
ocp4_install_baseDomain_resourcegroup: cs-rg
ocp4_install_resourcegroup: cs-rg-testcluster
ocp4_install_location: uksouth
ocp4_install_cluster_name: testcluster
ocp4_install_pullSecret: 'ADD PULL SECRET HERE'
ocp4_install_sshKey: "ADD PUB KEY HERE"

ocp4_install_master_replicas: 3
ocp4_install_worker_replicas: 3
ocp4_install_clusterNetwork_cidr: 10.128.0.0/14
ocp4_install_clusterNetwork_hostPrefix: 23
ocp4_install_machineNetwork_cidr: 10.0.0.0/16
ocp4_install_machineNetwork_networkType: OpenShiftSDN
ocp4_install_serviceNetwork_cidr: 172.30.0.0/16

ocp4_install_azure_it_master_diskSizeGB: 1024
ocp4_install_azure_it_master_instance_type: Standard_D8s_v3
ocp4_install_azure_it_worker_instance_type: Standard_D2s_v3
ocp4_install_azure_it_worker_diskSizeGB: 512

install_azure_it_worker_config_az_config:
  - name: baseDomainResourceGroupName
    value: "cs-rg"
  - name: region
    value: "uksouth"
  - name: resourceGroupName
    value: "cs-rg-testcluster"
  - name: outboundType
    value: "Loadbalancer"
  - name: cloudName
    value: "AzurePublicCloud"

ocp4_install_azure_it_worker_config_zones:
  - 1
  - 2
  - 3

ocp4_install_azure_subscriptionid: <REPLACE ME>
ocp4_install_azure_tenantid: <REPLACE ME>

ocp4_install_azure_clientid: <REPLACE ME>
ocp4_install_azure_clientsecret: <REPLACE ME>


```

License
-------

Apache License 2.0


Author Information
------------------
Ken Hitchcock [Github](https://github.com/kenhitchcock)

