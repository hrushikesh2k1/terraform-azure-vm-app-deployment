## Deplying the python flask application to azure vm using cloud init yaml + terraform

## pre-requisite
1. azure cli
2. terraform


## Terraform code
```
# main.tf
resource "azurerm_resource_group" "rg" {
  name     = "rg-terraform-vm"
  location = var.location
}

resource "azurerm_virtual_network" "vnet" {
  name                = "vm-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet" {
  name                 = "vm-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "pip" {
  name                = "vm-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}

resource "azurerm_network_interface" "nic" {
  name                = "vm-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

resource "azurerm_network_security_group" "nsg" {
  name                = "vm-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-SSH"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "python"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "8000"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface_security_group_association" "nsg_assoc" {
  network_interface_id      = azurerm_network_interface.nic.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}


resource "azurerm_linux_virtual_machine" "vm" {
  name                = "terraform-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2ats_v2"
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  disable_password_authentication = false

  depends_on = [
  azurerm_network_interface_security_group_association.nsg_assoc
]


  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  custom_data = base64encode(file("cloud-init.yaml"))

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}
```
```
# output.tf
output "public_ip" {
  value = azurerm_public_ip.pip.ip_address
}
```
```
# provider.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=4.1.0"
    }
  }
}

# Configure the Microsoft Azure Provider
provider "azurerm" {
  resource_provider_registrations = "none" # This is only required when the User, Service Principal, or Identity running Terraform lacks the permissions to register Azure Resource Providers.
  features {}
  subscription_id = ""
}
```
```
# variable.tf
 variable "location" {
  default = "Australia East"
}

variable "admin_username" {
  default = "azureuser"
}

variable "admin_password" {
  sensitive = true
}
```

## cloud-init.yaml file
```
#cloud-config

package_update: true

packages:
  - python3
  - python3-pip

write_files:
  - path: /opt/app.py
    permissions: '0644'
    content: |
      from flask import Flask

      app = Flask(__name__)

      @app.route("/")
      def hello():
          return "Hello from Cloud-init!"

      if __name__ == "__main__":
          app.run(host="0.0.0.0", port=8000)

runcmd:
  # install flask
  - pip3 install flask

  # move app to user's home AFTER user exists
  - mkdir -p /home/azureuser
  - mv /opt/app.py /home/azureuser/app.py
  - chown azureuser:azureuser /home/azureuser/app.py
```

## #cloud-config

Tells cloud-init to treat this file as configuration; without it, the file may be ignored.

## custom_data = base64encode(...)

Azure requires cloud-init data to be Base64 encoded before passing it to the VM.

## package_update: true

Runs apt update automatically at first boot to avoid package install failures.

## packages:

Installs OS packages during VM boot without SSH or provisioners.

## write_files

Creates files on the VM at boot time, replacing the need for SCP or Terraform file provisioners.

## write_files runs early

Files are written before user home directories may exist, so writing directly to /home/<user> can fail.

## Safe write location

Use paths like /opt or /var in write_files because they always exist.

## content: |

YAML literal block — preserves newlines exactly as written; required for multiline scripts or code.
