# Introduction
Purpose of this document is to outline how to create a Hedvig cluster within a cloud provider. This configuration is designed to provide engineers an education and demonstration environment. These configurations should not be used for production workloads.

# Prerequisites

This guide assumes the engineer is sufficiently familiar with the use of the following types of resources within the respective cloud provider
- Virtual networks
- Creating virtual machines / instances with block storage
- Firewall settings / security groups
- Resource groups (Azure)
- Storage accounts and containers (Azure)

# Disclaimer
Running cloud resources will incur costs and the engineer should validate the investment required to operate the cluster. "Message and data rates apply".

# Cloud Provider-specific Notes

## Azure
These are the specific installation instructions for creating the cluster in Azure. Presumes one has a subscription and sufficient privileges to create sufficient CPUs in the environment.

### Bucket creation to store software
1. Install [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html) and [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) on your local machine
2. Prepare for Terraform setup by creating a ```./terraform/azure/swbucket/terraform.tfvars``` file with your subscription id (assumes you've cloned the repo onto your own machine)
```
echo "subscription_id = \"<<your sub-id here>>\"\n" >> ./terraform/azure/swbucket/terraform.tfvars
```
3. Prepare for Ansible by creating a ```./ansible/vars.yaml``` file that looks like this replacing "<<name>>" with the name of the blob you will download
```
pwd: hedvig
jump_server:
software_filename: <<name>>
```
4. Create a resource group and bucket to store the Hedvig software zip file. This is separate from the resource group created in subsequent steps as resources in subsequent steps may be created / destroyed with regularity
```
cd ./ansible
ansible-playbook ./create_swbucket.yaml
```
5. Download the Hedvig software to your local machine
6. Upload the Hedvig software (zip file) to the previously created bucket

### Resource setup
1. Submit a service request to Azure to increase the number of CPUs to 64 for the subscription you will be using.
2. Prepare for Terraform setup by creating a ```./terraform/azure/resources/terraform.tfvars``` file with your subscription id (assumes you've cloned the repo onto your own machine)
```
echo "subscription_id = \"<<your sub-id here>>\"\n" >> ./terraform/azure/resources/terraform.tfvars
```
3. Generate a fresh set of SSH keys, create Azure resources, and capture the resulting jump server IP
```
ansible-playbook ./main0.yaml
```
4. Update the local known_hosts, validate connectivity, and prepare the VMs
```
ansible-playbook ./main1.yaml
```
5. Login to vm-deployment
```
export JUMP=`grep jump_server vars.yaml | awk '{split($0,a," "); print a[2]}'` && ssh -o UserKnownHostsFile=/dev/null -o GlobalKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -J azureuser@$JUMP azureuser@vm-deployment.internal.cloudapp.net
```
6. **On vm-deployment**, authenticate to Azure, and run remaining steps on the deployment server (via previously uploaded Ansible script)
```
su -l admin
az login
ansible-playbook /tmp/hedvig/main2.yaml
exit
```
7. Back on your localhost, add a window manager, xrdp, and google chrome to enable RDP into the machine. You will access the Hedvig web console from the jump server
```
ansible-playbook ./main3.yaml
```
8. Use RDP to login to the jump server
9. On the jump server, access the console (via /usr/bin/google-chrome) at http://vm-storagenode0.internal.cloudapp.net. Use ```azureuser``` as the username and the password you specified in ```vars.yaml``` as the password

### Resource destruction
Executing the follow step removes all Hedvig software and associated cloud resources. It is irreversible.   
```
ansible-playbook ./destroy_resources.yaml
```
### Software bucket destruction
Executing the follow step removes all Azure resources created with the ```create_swbucket.yaml``` playbook. It is irreversible.   
```
ansible-playbook ./destroy_swbucket.yaml
```