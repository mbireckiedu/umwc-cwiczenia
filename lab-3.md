# ☁️ Laboratorium 3 Terraform – Infrastruktura i Azure Machine Learning

## 🎯 Cele laboratorium

Po zakończeniu zajęć student potrafi:

* założyć i skonfigurować konto **Azure for Students**,
* uruchomić **Azure Cloud Shell** z kontem uczelnianym,
* zarejestrować wymagane **providery** dla Terraforma,
* utworzyć dwa niezależne projekty Terraform:  
  1️⃣ **terraform-vm-lab** – infrastruktura i maszyna wirtualna,  
  2️⃣ **terraform-aml-lab** – środowisko Azure Machine Learning Workspace,
* przygotować dokumentację z **Resource Visualizer**,
* oraz dodać całość do repozytorium GitHub.

## Spis treści

* [Cele laboratorium](#cele)
* [KROK 1 – Założenie konta Azure for Students](#krok1)
* [KROK 2 – Uruchomienie Azure Cloud Shell](#krok2)
* [KROK 3 – Sprawdzenie połączenia i aktywnej subskrypcji](#krok3)
* [KROK 4 – Rejestracja wymaganych providerów](#krok4)
* [KROK 5 – Utworzenie katalogu dla VM-lab](#krok5)
* [Wstęp do Terraform – przeczytaj zanim przejdziesz do kroku 6](#wstep)
* [KROK 6 – Plik `main.tf` (VM-lab)](#krok6)
* [KROK 7 – Uruchomienie VM-lab](#krok7)
* [KROK 8 – Utworzenie katalogu dla AML-lab](#krok8)
* [KROK 9 – Plik `main.tf` (AML-lab)](#krok9)
* [KROK 10 – Uruchomienie AML-lab](#krok10)
* [KROK 11 – Pliki README](#krok11)
* [KROK 12 – Dodanie projektów do repozytorium](#krok12)
* [KROK 13 – Naprawa środowiska Terraform w razie błędów](#naprawa)
* [Podsumowanie](#podsumowanie)

---

## 🧩 KROK 1 – Założenie konta Azure for Students
<a id="krok1"></a>
   🔹 Jeśli masz już konto w Azure dla domeny @dsw.edu.pl zaloguj się do portalu używając swoich danych. [https://portal.azure.com](https://portal.azure.com) 🔹
1. Przejdź na stronę:
   👉 [https://azure.microsoft.com/pl-pl/free/students](https://azure.microsoft.com/pl-pl/free/students)
2. Kliknij **„Rozpocznij bezpłatnie”**.
3. Zaloguj się kontem uczelnianym **@dsw.edu.pl**.
4. W formularzu:

   * **Country/Region:** Polska
   * **University:** Dolnośląska Szkoła Wyższa (DSW)
5. Po weryfikacji przejdź do [https://portal.azure.com](https://portal.azure.com).  
   Powinna być widoczna subskrypcja **Azure for Students**.
---

## ☁️ KROK 2 – Uruchomienie Azure Cloud Shell
<a id="krok2"></a>
1. Kliknij ikonę **>_ (Cloud Shell)** w prawym górnym rogu portalu.
2. Wybierz powłokę **Bash**.
3. Jeśli pojawi się komunikat *“You have no storage mounted”*, kliknij **Create storage** i wybierz. Podczas tego kroku może wystąpić błąd. Jeśli tak będzie skorzystaj z opcji Cloud shell bez Storage Account:

   * **Subscription:** Azure for Students
   * **Resource group:** `rg-cloudshell`
   * **Region:** `France Central`
   * **Storage Account:** np. `cloudshellstor<twojeinicjały>`
   * **File Share:** `cloudshell`
4. Poczekaj, aż środowisko się załaduje.

---

## 🔐 KROK 3 – Sprawdzenie połączenia i aktywnej subskrypcji
<a id="krok3"></a>

W Cloud Shell wpisz:

```bash
az account show --query "{Name:name, User:user.name, SubscriptionId:id, TenantId:tenantId, IsDefault:isDefault, State:state}" --output table
```

Przykładowy wynik:

| Name               | CloudName  | SubscriptionId                       | State   | IsDefault | User                                                      |
| ------------------ | ---------- | ------------------------------------ | ------- | --------- | --------------------------------------------------------- |
| Azure for Students | AzureCloud | 495e64a3-1009-4ed0-b14a-38a5456c2720 | Enabled | True      | [jan.kowalski@dsw.edu.pl](mailto:jan.kowalski@dsw.edu.pl) |

✅ Upewnij się, że:

* **Name** = Azure for Students
* **User** to Twój e-mail uczelniany
* **State** = Enabled

Jeśli nie:

```bash
az account set --subscription "Azure for Students"
```

---

## 🧰 KROK 4 – Rejestracja wymaganych providerów
<a id="krok4"></a>

W portalu Azure:

1. Przejdź do: **Subscriptions → [Twoja subskrypcja] → Resource providers**
2. Wyszukaj i kliknij **Register** dla:

| Provider                          | Opis                         |
| --------------------------------- | ---------------------------- |
| Microsoft.Network                 | sieci i adresy IP            |
| Microsoft.Compute                 | maszyny wirtualne            |
| Microsoft.Storage                 | konta Storage                |
| Microsoft.KeyVault                | sejf sekretów                |
| Microsoft.ContainerRegistry       | rejestr kontenerów           |
| Microsoft.MachineLearningServices | AML workspace                |
| Microsoft.OperationalInsights     | Log Analytics                |
| Microsoft.Insights                | Application Insights         |
| **Microsoft.AzureTerraform**      | integracja Terraform z Azure |

---

## 🧱 KROK 5 – Utworzenie katalogu dla VM-lab
<a id="krok5"></a>

W Cloud Shell:

```bash
cd ~
mkdir terraform-vm-lab && cd terraform-vm-lab
```

---

## 📚 Wstęp do Terraform – przeczytaj zanim przejdziesz do kroku 6
<a id="wstep"></a>

## 🧠 Opis pracy z Terraformem – krok po kroku


Terraform to narzędzie typu **Infrastructure as Code (IaC)**,
czyli służące do opisywania infrastruktury w postaci kodu –
zamiast klikać zasoby w portalu Azure, definiujemy je w plikach `.tf`.

### 🔹 1. Struktura projektu Terraform

Każdy projekt Terraform (tzw. *workspace*) zawiera:

* **plik konfiguracyjny `.tf`** – opis zasobów, jakie chcemy utworzyć,
* **plik stanu (`terraform.tfstate`)** – zapis aktualnego stanu infrastruktury,
* **folder `.terraform`** – pobrane providery (np. `azurerm`).

Każdy katalog, w którym wykonamy `terraform init`, staje się **oddzielnym workspace’em**.
Dlatego w naszych laboratoriach tworzymy **dwa katalogi**: `terraform-vm-lab` i `terraform-aml-lab` – aby rozdzielić środowiska i ich stan.

### 🔹 2. Plik `main.tf`

To główny plik projektu – jego rola to:

* zdefiniowanie **providerów** (połączenie z Azure),
* ustawienie **lokalnych zmiennych** (np. `student_id`, `location`),
* opis zasobów (`resource`),
* **wyjścia (`output`)** – wartości wyświetlane po wdrożeniu (np. publiczny IP).

Każdy blok `resource` odpowiada jednemu elementowi w Azure —
np. `azurerm_virtual_network` tworzy sieć, `azurerm_linux_virtual_machine` – maszynę,
a `azurerm_machine_learning_workspace` – środowisko uczenia maszynowego.

### 🔹 3. Provider `azurerm`

Provider to wtyczka, która pozwala Terraformowi komunikować się z Azure:

```hcl
provider "azurerm" {
  features {}
}
```

Terraform użyje poświadczeń aktywnego użytkownika Cloud Shell i będzie tworzyć zasoby w Twojej subskrypcji.

### 🔹 4. Zmienna `locals` i numer indeksu

Blok `locals` zawiera **lokalne zmienne**, np. lokalizację i numer indeksu:

```hcl
locals {
  student_id = "12345"
  location   = "francecentral"
}
```

Dzięki temu każda nazwa zasobu ma unikalny sufiks, np. `vm-12345`.

### 🔹 5. Główne komendy Terraform

#### ✅ `terraform init`

* Inicjuje projekt, pobiera **providery** (np. `azurerm`, `random`), tworzy `.terraform/`.
* **Bez tego polecenia żadne inne nie zadziała.**

#### ✅ `terraform plan`

* Analizuje kod `.tf` vs. stan w Azure i wyświetla **plan zmian** (symulacja).
* Niczego nie tworzy – pozwala zweryfikować, co się wydarzy.

#### ✅ `terraform apply`

* Wykonuje plan i **tworzy zasoby**.
* Po zakończeniu pokazuje **outputs** (np. IP maszyny, link do AML Studio).
* Flaga `-auto-approve` pomija pytanie o potwierdzenie (wygodne na labie).

#### ✅ `terraform output`

* Pokazuje wartości zdefiniowane jako `output`, np.:

  ```bash
  terraform output -raw public_ip
  ```
* Możesz użyć ich w dalszych komendach (np. SSH).

#### ✅ `terraform destroy`

* Usuwa wszystkie zasoby utworzone przez Terraform (wg `terraform.tfstate`).
* **Zawsze** wykonuj `destroy` na końcu labu, by nie zużywać limitów Azure for Students.

---

Skoro wiemy, jak działa Terraform, przechodzimy do tworzenia pierwszego pliku `main.tf`.

---

## 📄 KROK 6 – Plik `main.tf` (VM-lab)
<a id="krok6"></a>

Zapoznaj się z edytorem Cloud Shell:
[https://learn.microsoft.com/en-us/azure/cloud-shell/using-cloud-shell-editor](https://learn.microsoft.com/en-us/azure/cloud-shell/using-cloud-shell-editor)
W Cloud Shell można też korzystać z edytorów `nano` albo `vi/vim`.

Stwórz plik `main.tf` w katalogu `terraform-vm-lab`.

Każdy student uzupełnia w sekcji `locals` swój **numer indeksu**:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

locals {
  student_id = "12345" # <<< WPISZ SWÓJ NUMER INDEKSU
  location   = "francecentral"
  rg_name    = "rg-lab-${local.student_id}"
  vnet_name  = "vnet-${local.student_id}"
  subnet     = "snet-${local.student_id}"
  pip_name   = "pip-${local.student_id}"
  nsg_name   = "nsg-${local.student_id}"
  nic_name   = "nic-${local.student_id}"
  vm_name    = "vm-${local.student_id}"
  admin_user = "azureuser"
  admin_pass = "LabPassword123!"
}

resource "azurerm_resource_group" "lab" {
  name     = local.rg_name
  location = local.location
}

resource "azurerm_virtual_network" "lab" {
  name                = local.vnet_name
  location            = local.location
  resource_group_name = azurerm_resource_group.lab.name
  address_space       = ["10.10.0.0/16"]
}

resource "azurerm_subnet" "lab" {
  name                 = local.subnet
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = ["10.10.1.0/24"]
}

resource "azurerm_public_ip" "lab" {
  name                = local.pip_name
  location            = local.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_security_group" "lab" {
  name                = local.nsg_name
  location            = local.location
  resource_group_name = azurerm_resource_group.lab.name

  security_rule {
    name                       = "Allow-SSH-22"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface" "lab" {
  name                = local.nic_name
  location            = local.location
  resource_group_name = azurerm_resource_group.lab.name

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.lab.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.lab.id
  }
}

resource "azurerm_network_interface_security_group_association" "lab" {
  network_interface_id      = azurerm_network_interface.lab.id
  network_security_group_id = azurerm_network_security_group.lab.id
}

resource "azurerm_linux_virtual_machine" "lab" {
  name                = local.vm_name
  resource_group_name = azurerm_resource_group.lab.name
  location            = local.location
  size                = "Standard_B1s"
  admin_username      = local.admin_user
  admin_password      = local.admin_pass
  disable_password_authentication = false
  network_interface_ids = [azurerm_network_interface.lab.id]

  os_disk {
    name                 = "osdisk-${local.student_id}"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}

output "public_ip" {
  value = "ssh azureuser@${azurerm_public_ip.lab.ip_address}"
}
```

W outputach znajdziesz komendę do SSH.

Sprawdź, czy możesz zalogować się poprawnie do VM.

**Podpowiedź:** jeśli chcesz wrócić z VM do Cloud Shell, w terminalu VM wpisz: `exit`.

---

## ▶️ KROK 7 – Uruchomienie VM-lab
<a id="krok7"></a>

```bash
terraform init
terraform plan
terraform apply -auto-approve
terraform output -raw public_ip
```

📸 **Zrób zrzut ekranu z Resource Visualizer przed `terraform destroy`.**  
Ścieżka: **Resource groups → rg-lab-<nr_indeksu> → Resource visualizer**  
Możesz też użyć opcji **Export Resource Visualizer** do pliku `.png`.

Następnie:

```bash
terraform destroy -auto-approve
```

---

## 🧠 KROK 8 – Utworzenie katalogu dla AML-lab
<a id="krok8"></a>

```bash
cd ~
mkdir terraform-aml-lab && cd terraform-aml-lab
```

---

## 📄 KROK 9 – Plik `main.tf` (AML-lab)
<a id="krok9"></a>

Stwórz plik `main.tf` w katalogu `terraform-aml-lab`.

Każdy student uzupełnia w sekcji `locals` swój **numer indeksu**:

```hcl
############################
# Azure Machine Learning – pełny zestaw zależności
# Region: francecentral
############################

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

# Dane o kliencie/subskrypcji – potrzebne do Key Vault i wygodnych outputów (URL)
data "azurerm_client_config" "current" {}
data "azurerm_subscription" "current" {}

# ---- Lokalne nazwy zasobów ----
locals {
  student_id    = "111111" # <<< WPISZ SWÓJ NUMER INDEKSU
  location      = "francecentral"
  rg_name       = "rg-aml-${local.student_id}"
  sa_name       = "stor${local.student_id}"
  kv_name       = "kv${local.student_id}"
  la_name       = "la${local.student_id}"
  ai_name       = "ai${local.student_id}"
  acr_name      = "acr${local.student_id}"
  aml_ws_name   = "amlws-${local.student_id}"
}

# Generator losowego suffiksu do Storage Account (unikalność DNS)
resource "random_string" "sa_suffix" {
  length  = 6
  upper   = false
  lower   = true
  numeric = true
  special = false
}

resource "azurerm_resource_group" "aml" {
  name     = local.rg_name
  location = local.location
}

# ---- Storage Account dla AML ----
resource "azurerm_storage_account" "aml" {
  name                               = local.sa_name
  resource_group_name                = azurerm_resource_group.aml.name
  location                           = local.location
  account_tier                       = "Standard"
  account_replication_type           = "LRS"
  allow_nested_items_to_be_public    = false
  min_tls_version                    = "TLS1_2"

  blob_properties {
    versioning_enabled = true
  }

  lifecycle {
    prevent_destroy = false
  }
}

# ---- Log Analytics Workspace (monitoring) ----
resource "azurerm_log_analytics_workspace" "aml" {
  name                = local.la_name
  location            = local.location
  resource_group_name = azurerm_resource_group.aml.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# ---- Application Insights (workspace-based) ----
resource "azurerm_application_insights" "aml" {
  name                = local.ai_name
  location            = local.location
  resource_group_name = azurerm_resource_group.aml.name
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.aml.id
}

# ---- Key Vault (sekrety AML) ----
resource "azurerm_key_vault" "aml" {
  name                              = local.kv_name
  resource_group_name               = azurerm_resource_group.aml.name
  location                          = local.location
  tenant_id                         = data.azurerm_client_config.current.tenant_id
  sku_name                          = "standard"
  purge_protection_enabled          = false
  soft_delete_retention_days        = 7
  enabled_for_deployment            = true
  enabled_for_template_deployment   = true
  public_network_access_enabled     = true
}

# Dostęp do Key Vault dla aktualnego użytkownika
resource "azurerm_key_vault_access_policy" "current_user" {
  key_vault_id = azurerm_key_vault.aml.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = data.azurerm_client_config.current.object_id

  secret_permissions = ["Get", "List", "Set", "Delete", "Purge", "Recover", "Backup","Restore"]
}

# ---- Azure Container Registry ----
resource "azurerm_container_registry" "aml" {
  name                = local.acr_name
  resource_group_name = azurerm_resource_group.aml.name
  location            = local.location
  sku                 = "Basic"
  admin_enabled       = true
}

# ---- Azure Machine Learning Workspace (v2) ----
resource "azurerm_machine_learning_workspace" "aml" {
  name                    = local.aml_ws_name
  location                = local.location
  resource_group_name     = azurerm_resource_group.aml.name

  application_insights_id = azurerm_application_insights.aml.id
  key_vault_id            = azurerm_key_vault.aml.id
  storage_account_id      = azurerm_storage_account.aml.id
  container_registry_id   = azurerm_container_registry.aml.id

  identity {
    type = "SystemAssigned"
  }

  public_network_access_enabled = true
  description                   = "AML workspace for lab (francecentral)"
}

# ---- Outputs ----
locals {
  aml_ws_resource_id = azurerm_machine_learning_workspace.aml.id
  aml_portal_url     = "https://ml.azure.com/?wsid=${local.aml_ws_resource_id}"
}

output "aml_workspace_name" {
  value       = azurerm_machine_learning_workspace.aml.name
  description = "Nazwa AML Workspace"
}

output "aml_workspace_id" {
  value       = local.aml_ws_resource_id
  description = "Resource ID AML Workspace"
}

output "aml_portal_url" {
  value       = "Link do AML w przeglądarce: ${local.aml_portal_url}"
  description = "Szybkie przejście do AML w przeglądarce"
}
```

> **Uwaga:** jeśli w regionie/uczelnianej polityce aktualny rozmiar VM lub zasób nie jest dozwolony, zmień `location` lub SKU zgodnie z komunikatem błędu.

---

## ▶️ KROK 10 – Uruchomienie AML-lab
<a id="krok10"></a>

```bash
terraform init
terraform plan
terraform apply -auto-approve
terraform output
```

W outputach znajdziesz link do AML w przeglądarce. Sprawdź, czy jest poprawny i czy można do niego wejść.

📸 Wykonaj screenshot z **Resource Visualizer** **przed** `terraform destroy`  
(**Resource groups → rg-aml-<nr_indeksu> → Resource visualizer**)  
Możesz też użyć opcji **Export Resource Visualizer** do pliku `.png`.

Następnie:

```bash
terraform destroy -auto-approve
```

---

## 📁 KROK 11 – Pliki README
<a id="krok11"></a>

### `terraform-vm-lab/README.md`

```markdown
# Terraform Lab – Maszyna wirtualna w Azure

Ten folder zawiera kod Terraform, który tworzy kompletną infrastrukturę sieciową  
oraz maszynę wirtualną Linux (Ubuntu 22.04 LTS) w regionie France Central.  
Wszystkie nazwy zasobów zawierają numer indeksu studenta.

## Utworzone zasoby
- Resource Group  
- Virtual Network  
- Subnet  
- Public IP  
- Network Security Group (z regułą SSH)  
- Network Interface  
- Linux Virtual Machine  

## Architektura
![Zrzut ekranu z Resource Visualizer](./resource-visualizer.png)
```

### `terraform-aml-lab/README.md`

```markdown
# Terraform Lab – Azure Machine Learning

Ten folder zawiera kod Terraform, który tworzy środowisko Azure Machine Learning Workspace  
wraz ze wszystkimi zależnościami (Storage, Key Vault, Log Analytics, Application Insights, ACR)  
w regionie France Central.  
Wszystkie nazwy zasobów zawierają numer indeksu studenta.

## Utworzone zasoby
- Storage Account (dla AML)  
- Key Vault  
- Log Analytics Workspace  
- Application Insights  
- Container Registry  
- AML Workspace  

## Architektura
![Zrzut ekranu z Resource Visualizer](./resource-visualizer.png)
```

---

## 🧾 KROK 12 – Dodanie projektów do repozytorium
<a id="krok12"></a>

W repozytorium GitHub utwórz katalog `terraform/` i skopiuj oba projekty:

```
terraform/
├── terraform-vm-lab/
│   ├── main.tf
│   ├── README.md
│   └── resource-visualizer.png
└── terraform-aml-lab/
    ├── main.tf
    ├── README.md
    └── resource-visualizer.png
```

Commit i push:

```bash
git add terraform/
git commit -m "Dodano laboratoria Terraform (VM + AML) z dokumentacją"
git push origin main
```

---

## 🛟 Naprawa środowiska Terraform w razie błędów
<a id="naprawa"></a>

Jeśli podczas `apply`/`destroy` pojawią się błędy (np. przerwana sesja, niedozwolony region, częściowo utworzone zasoby) i Terraform „gubi” stan:

1. **Usuń Resource Group ręcznie w portalu Azure**

   * Portal → **Resource groups** → wybierz grupę (np. `rg-lab-<nr>` lub `rg-aml-<nr>`) → **Delete resource group**
   * Potwierdź nazwę i poczekaj na usunięcie wszystkich zasobów.

2. **Usuń lokalny stan Terraform w Cloud Shell**
   Wróć do katalogu domowego i usuń katalogi workspace’ów:

   ```bash
   cd ~
   rm -rf terraform-vm-lab terraform-aml-lab
   ```

   (Usuwa to `.terraform/` i `terraform.tfstate`, czyli błędny stan.)

3. **Zacznij od nowa**
   Wróć do kroków 5 i 6 i uruchom `terraform init` `terraform plan` i `terraform apply`.



   To przywróci czyste, spójne środowisko.


---

## ✅ Podsumowanie
<a id="podsumowanie"></a>

| Etap                              | Rezultat               |
| --------------------------------- | ---------------------- |
| Konto Azure for Students          | Aktywne                |
| Cloud Shell                       | Skonfigurowane         |
| Provider Microsoft.AzureTerraform | Zarejestrowany         |
| VM-lab                            | Utworzony i zniszczony |
| AML-lab                           | Utworzony i zniszczony |
| README + Screenshot               | Uzupełnione            |
| Repozytorium GitHub               | Zaktualizowane         |

---
