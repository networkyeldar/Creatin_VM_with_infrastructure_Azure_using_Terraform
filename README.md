# Creating_VM_with_infrastructure_Azure_using_Terraform

Terraform allows you to define and create complete infrastructure deployments in Azure. This instruction shows you how to createa complete Linux environment and supporting resources with Terraform

# Create an Azure connection and resource group

```
provider "azurerm" {
# The "feature" block is required for AzureRM provider 2.x.
# If you're using version 1.x, the "features" block is not allowed.
version = "~>2.0"
features {}
}
```

```
resource "azurerm_resource_group" "myterraformgroup" {
name = "myResourceGroup"
location = "eastus"
tags = {
environment = "Terraform Demo"
}
}
```

# Create virtual network
The following section creates a virtual network named myVnet in the 10.0.0.0/16 address space:

```
resource "azurerm_virtual_network" "myterraformnetwork" {
name = "myVnet"
address_space = ["10.0.0.0/16"]
location = "eastus"
resource_group_name = azurerm_resource_group.myterraformgroup.name
tags = {
environment = "Terraform Demo"
}
}
```

The following section creates a subnet named ```mySubnet``` in the ```myVnet``` virtual network:

```
resource "azurerm_subnet" "myterraformsubnet" {
name = "mySubnet"
resource_group_name = azurerm_resource_group.myterraformgroup.name
virtual_network_name = azurerm_virtual_network.myterraformnetwork.name
address_prefixes = ["10.0.2.0/24"]
}
```

# Create public IP address
```
resource "azurerm_public_ip" "myterraformpublicip" {
name = "myPublicIP"
location = "eastus"
resource_group_name = azurerm_resource_group.myterraformgroup.name
allocation_method = "Dynamic"
tags = {
environment = "Terraform Demo"
}
}
```

# Create Network Security Group
Network Security Groups control the flow of network traffic in and out of your VM.The following section creates
a network security group named ```myNetworkSecurityGroup``` and defines a rule to allow SSH traffic on TCP port 22:
```
resource "azurerm_network_security_group" "myterraformnsg" {
name = "myNetworkSecurityGroup"
location = "eastus"
resource_group_name = azurerm_resource_group.myterraformgroup.name
security_rule {
name = "SSH"
priority = 1001
direction = "Inbound"
access = "Allow"
protocol = "Tcp"
source_port_range = "*"
destination_port_range = "22"
source_address_prefix = "*"
destination_address_prefix = "*"
}
tags = {
environment = "Terraform Demo"
}
}
```
# Create virtual network interface card
A virtual network interfacecard (NIC) connects your VM to a given virtual network, public IP address,and
network security group.The following section in a Terraform template creates a virtual NIC named myNIC
connected to thevirtual networking resources you've created:
```
resource "azurerm_network_interface" "myterraformnic" {
name = "myNIC"
location = "eastus"
resource_group_name = azurerm_resource_group.myterraformgroup.name
ip_configuration {
name = "myNicConfiguration"
subnet_id = azurerm_subnet.myterraformsubnet.id
private_ip_address_allocation = "Dynamic"
public_ip_address_id = azurerm_public_ip.myterraformpublicip.id
}
tags = {
environment = "Terraform Demo"
}
}
```
# Connect the security group to the network interface
```
resource "azurerm_network_interface_security_group_association" "example" {
network_interface_id = azurerm_network_interface.myterraformnic.id
network_security_group_id = azurerm_network_security_group.myterraformnsg.id
}
```
# Create storage account for diagnostics
To store boot diagnostics for a VM,you need a storage a ccount.These boot diagnostics can help you
troubleshoot problems and monitor thestatus of your VM.
```
resource "random_id" "randomId" {
keepers = {
#Generate a new ID only when a new resource group is defined
resource_group = azurerm_resource_group.myterraformgroup.name
}
byte_length = 8
}
```
Now you can create a storage account.The following section creates a storage account, with the name based on
the random text generated in the preceding step:
```
resource "azurerm_storage_account" "mystorageaccount" {
name = "diag${random_id.randomId.hex}"
resource_group_name = azurerm_resource_group.myterraformgroup.name
location = "eastus"
account_replication_type = "LRS"
account_tier = "Standard"
tags = {
environment = "Terraform Demo"
}
}
```
# Create virtual machine

The final step is to create a VM and use all the resources created.The following section creates a VM named
```myVM``` and attaches the virtual NIC named ```myNIC``` .
SSH key datais provided in the ```ssh_keys``` section. Provide a public SSH key in the ```key_data``` field.
```
resource "tls_private_key" "example_ssh" {
algorithm = "RSA"
rsa_bits = 4096
}
```
```
output "tls_private_key" { value = tls_private_key.example_ssh.private_key_pem }
resource "azurerm_linux_virtual_machine" "myterraformvm" {
name = "myVM"
location = "eastus"
resource_group_name = azurerm_resource_group.myterraformgroup.name
network_interface_ids = [azurerm_network_interface.myterraformnic.id]
size = "Standard_DS1_v2"
os_disk {
name = "myOsDisk"
caching = "ReadWrite"
storage_account_type = "Premium_LRS"
}
source_image_reference {
publisher = "Canonical"
offer = "UbuntuServer"
sku = "18.04-LTS"
version = "latest"
}
computer_name = "myvm"
admin_username = "azureuser"
disable_password_authentication = true
admin_ssh_key {
username = "azureuser"
public_key = tls_private_key.example_ssh.public_key_openssh
}
boot_diagnostics {
storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
}
tags = {
environment = "Terraform Demo"
}
}
```
# CompleteTerraformscript
Bring all these sections together and see Terraform in action,create a file called ```terraform_azure.tf```.

# Build and deploy the infrastructure
With your Terraform template created, the first step is to initialize Terraform.This step ensures that Terraform
has all the prerequisites to build your templatein Azure
```terraform init```

The next step is to have Terraform review and validate the template.This step compares the requested resources
to the state information saved by Terraform and then outputs the planned execution.The Azure resources aren't
created at this point
```terraform plan```

If everything looks correct and you're ready to build the infrastructure in Azure,apply the template in Terraform:
```terraform apply```

Once Terraform completes,your VM infrastructure is ready. Obtain the public IP address of your VM with az vm
show:

```az vm show --resource-group myResourceGroup --name myVM -d --query [publicIps] -o tsv```

You can then SSH to your VM:
```ssh azureuser@<publicIps>```



