
Para implementar máquinas virtuales en Azure utilizando Terraform desde un entorno Linux, sigue estos pasos detallados:

## 1. Instalación de Prerrequisitos

### a. Azure CLI

La Azure CLI es una herramienta de línea de comandos que facilita la gestión de recursos en Azure.

- **Instalación en Ubuntu**:

  Abre una terminal y ejecuta los siguientes comandos:

  ```bash
  curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
  ```

  Este comando descarga e instala la Azure CLI en tu sistema.

### b. Terraform

Terraform es una herramienta de infraestructura como código que permite definir y gestionar la infraestructura de manera declarativa.

- **Instalación en Ubuntu**:

  1. Añade el repositorio oficial de HashiCorp:

     ```bash
     sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
     wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
     echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
     ```

  2. Actualiza los repositorios e instala Terraform:

     ```bash
     sudo apt-get update
     sudo apt-get install terraform
     ```

### c. Visual Studio Code (VS Code)

VS Code es un editor de código fuente que facilita la edición y gestión de archivos de configuración.

- **Instalación en Ubuntu**:

  Si ya tienes VS Code instalado desde la Microsoft Store, puedes omitir este paso.

  Para instalar VS Code desde la terminal:

  ```bash
  sudo snap install --classic code
  ```

## 2. Configuración de Azure CLI

Autentícate en tu cuenta de Azure utilizando la Azure CLI:

```bash
az login
```

Este comando abrirá una ventana del navegador para que ingreses tus credenciales de Azure.

## 3. Creación de una Entidad de Servicio (Service Principal)

Terraform requiere permisos para gestionar recursos en Azure. Para ello, crea una entidad de servicio:

```bash
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/<ID_DE_TU_SUSCRIPCIÓN>"
```

Reemplaza `<ID_DE_TU_SUSCRIPCIÓN>` con el ID de tu suscripción de Azure, que puedes obtener ejecutando `az account show`.

Este comando proporcionará un `appId`, `password` y `tenant`.

## 4. Configuración de Variables de Entorno

Establece las siguientes variables de entorno con los valores obtenidos en el paso anterior:

```bash
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_SUBSCRIPTION_ID="<ID_DE_TU_SUSCRIPCIÓN>"
export ARM_TENANT_ID="<tenant>"
```

Reemplaza los valores entre `< >` con la información correspondiente.

## 5. Creación de Archivos de Configuración de Terraform

Crea un directorio para tu proyecto y navega a él:

```bash
mkdir terraform-azure-vm
cd terraform-azure-vm
```

Dentro de este directorio, crea los siguientes archivos:

### a. `main.tf`

Este archivo define los recursos que se crearán en Azure:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "myResourceGroup"
  location = "East US"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "myVnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet" {
  name                 = "mySubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "nic" {
  name                = "myNIC"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "myVM"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_DS1_v2"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }
}
```

### b. `variables.tf`

Define variables para reutilización y flexibilidad:

```hcl
variable "resource_group_name" {
  description = "Nombre del grupo de recursos"
  default     = "myResourceGroup"
}

variable "location" {
  description = "Ubicación de los recursos en Azure"
  default     = "East US"
}

variable "admin_username" {
  description = "Nombre de usuario administrador para la VM"
  default     = " 
