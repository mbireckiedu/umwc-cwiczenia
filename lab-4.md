# 🧪 Laboratorium 4 Terraform + Azure ML

*Terraform + Azure ML CLI v2, Cloud Shell*

---

## Spis treści

* [Cele laboratorium](#cele)
* [KROK 0 — PAT i klon repozytorium](#krok0)
* [KROK 1 — Infrastruktura AML z Terraform](#krok1)
* [KROK 2 — Konfiguracja Azure ML CLI v2](#krok2)
* [KROK 3 — Klaster obliczeniowy (Compute)](#krok3)
* [KROK 4 — Kod eksperymentu](#krok4)
* [KROK 5 — Definicja joba (YAML) — sprawdzona wersja środowiska](#krok5)
* [KROK 6 — Uruchomienie i logi](#krok6)
* [KROK 7 — Wyniki (Studio + pobieranie artefaktów)](#krok7)
* [KROK 7a — Pobranie artefaktów z joba](#krok7a)
* [KROK 7b — Dodanie artefaktów do repo (`aml-lab-01/outputs`)](#krok7b)
* [KROK 7c — Rejestracja (publikacja) modelu w AML](#krok7c)
* [KROK 7d — Weryfikacja modelu w AML Studio](#krok7d)
* [KROK 8 — Sprzątanie zasobów](#krok8)
* [KROK 9 — Dokumentacja i commit do repo](#krok9)
* [Dokumentacja (Microsoft/GitHub)](#docs)

---

## 🎯 Cele laboratorium <a id="cele"></a>

* Połączyć się z repozytorium GitHub w **Cloud Shell** (PAT).
* Utworzyć **AML Workspace** Terraformem (RG, SA, KV, ACR, LA, AppInsights).
* Skonfigurować **Azure ML CLI v2** i utworzyć **Compute Cluster**.
* Uruchomić **job** (scikit-learn) i obejrzeć wyniki w **Azure ML Studio**.
* **Pobrać artefakty**, dodać do repo i **zarejestrować model** jako asset AML.
* Poprawnie posprzątać zasoby (compute + Terraform destroy).

---

## 🔐 KROK 0 — PAT i klon repozytorium <a id="krok0"></a>

### 0.1 GitHub PAT

1. GitHub → **Profile → Settings → Developer settings → Personal access tokens**
2. Wybierz **Fine-grained token**
3. Dostęp do *tego* repo (Contents – read/write), **Expiration** krótka (np. 7 dni)
4. Jeśli Twoja organizacja wymaga SSO, **autoryzuj token** dla organizacji

---

### 0.2 Klon w Azure Cloud Shell

```bash
git config --global user.name  "Imię Nazwisko"
git config --global user.email "twoj.mail@dsw.edu.pl"
git config --global credential.helper 'cache --timeout=3600'

git clone https://github.com/<owner>/<repo>.git
cd <repo>
```

> Podczas klonowania: **Username =** login GitHub, **Password =** *PAT*

---

### 0.3 Dodaj `.gitignore`

W głównym katalogu repozytorium stwórz plik `.gitignore` z zawartością:

```text
# Local .terraform directories
.terraform/

# .tfstate files
*.tfstate
*.tfstate.*
```

---

## ⚙️ KROK 1 — Infrastruktura AML z Terraform <a id="krok1"></a>

Przejdź do katalogu Twojego repozytorium i znajdź katalog `terraform/terraform-aml-lab`:

```bash
cd ~/<nazwa-twojego-repozytorium>/terraform/terraform-aml-lab
```

### 1.1 Potwierdź subskrypcję

```bash
az account show --query "{Name:name, User:user.name, SubscriptionId:id, TenantId:tenantId, IsDefault:isDefault, State:state}" --output table
az account set --subscription "Azure for Students"
```

---

### 1.2 Uruchom Terraform

```bash
terraform init
terraform plan
terraform apply -auto-approve
terraform output
```

Po zakończeniu zrób **screenshot Resource Visualizer** z portalu Azure
(*Resource groups → rg-aml-<indeks> → Resource visualizer*).

---

## ☁️ KROK 2 — Konfiguracja Azure ML CLI v2 <a id="krok2"></a>

```bash
az extension add -n ml -y
az configure --defaults group=rg-aml-<indeks> workspace=amlws-<indeks> location=francecentral
```

---

## 🧮 KROK 3 — Klaster obliczeniowy (Compute) <a id="krok3"></a>

```bash
CLUSTER=cpu-cluster-<indeks>

az ml compute create \
  --name $CLUSTER \
  --type amlcompute \
  --size Standard_DS2_v2 \
  --min-instances 0 \
  --max-instances 1 \
  --idle-time-before-scale-down 120
```

> Klaster skaluje się do zera po bezczynności (oszczędza darmowe limity).

---

## 🧰 KROK 4 — Kod eksperymentu <a id="krok4"></a>

W Cloud Shell:

```bash
mkdir -p ~/<nazwa-twojego-repozytorium>/aml-lab-01/src
cd ~/<nazwa-twojego-repozytorium>/aml-lab-01
```

**`src/train.py`**

```python
import json
from pathlib import Path
from sklearn.datasets import make_classification
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import joblib

def main():
    X, y = make_classification(n_samples=1000, n_features=20, n_informative=6, random_state=42)
    Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.25, random_state=42)

    clf = LogisticRegression(max_iter=1000)
    clf.fit(Xtr, ytr)
    acc = accuracy_score(yte, clf.predict(Xte))

    out = Path("outputs")
    out.mkdir(exist_ok=True)
    (out / "metrics.json").write_text(json.dumps({"accuracy": acc}, indent=2))
    joblib.dump(clf, out / "model.joblib")
    print(f"Model zapisany. Dokładność: {acc:.4f}")

if __name__ == "__main__":
    main()
```

---

## 📄 KROK 5 — Definicja joba (YAML) — sprawdzona wersja środowiska <a id="krok5"></a>

Używamy **środowiska** z rejestru `azureml`: `sklearn-1.5`, **wersja 34**.

**`job.yml`**

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
type: command
display_name: intro-sklearn-job
experiment_name: intro-sklearn
code: .
command: python src/train.py
environment: azureml://registries/azureml/environments/sklearn-1.5/versions/34
compute: azureml:cpu-cluster-<indeks>
limits:
  timeout: 3600
```

### Co się dzieje przy uruchomieniu joba?

1. **CLI pakuje katalog (`code: .`)** i wysyła go do AML Workspace.
2. AML uruchamia **kontener z bibliotekami** (`scikit-learn`, `numpy`, `pandas`, itd.).
3. Kod `train.py` jest wykonywany w tym środowisku.
4. Wszystko zapisane w `outputs/` jest automatycznie **zarchiwizowane jako artefakty joba**.
5. Możesz to zobaczyć w **Azure ML Studio → Jobs → intro-sklearn → Run**.

---

## ▶️ KROK 6 — Uruchomienie i logi <a id="krok6"></a>

```bash
JOB_ID=$(az ml job create --file job.yml --query name -o tsv)
az ml job stream --name $JOB_ID
```

### Co się dzieje w tle?

1. Twój kod i pliki konfiguracyjne są wysyłane do AML Workspace.
2. AML przypisuje jobowi unikalny **JOB_ID** i uruchamia go na klastrze.
3. Job przechodzi przez fazy: *Queued → Preparing → Running → Completed*.
4. `az ml job stream` pokazuje logi w czasie rzeczywistym, aż job się zakończy.
5. Po zakończeniu możesz przejść do portalu, aby obejrzeć wyniki i metryki.

---

## 🔎 KROK 7 — Wyniki (Studio + pobieranie artefaktów) <a id="krok7"></a>

**W Azure ML Studio**

* Otwórz **Studio** (link z `terraform output`), przejdź:
  *Jobs → intro-sklearn → ostatni Run*.
* Sprawdź zakładki **Outputs + logs** (`model.joblib`, `metrics.json`).

---

## 🧷 KROK 7a — Pobranie artefaktów z joba <a id="krok7a"></a>

```bash
mkdir -p ~/aml-lab-01-results/run_download
az ml job download --name $JOB_ID --download-path ~/aml-lab-01-results/run_download
ls -R ~/aml-lab-01-results/run_download
# najczęściej pliki będą w:
# ~/aml-lab-01-results/run_download/artifacts/outputs/model.joblib
# ~/aml-lab-01-results/run_download/artifacts/outputs/metrics.json
```

---

## 📦 KROK 7b — Dodanie artefaktów do repo (`aml-lab-01/outputs`) <a id="krok7b"></a>

```bash
mkdir -p ~/<nazwa-twojego-repozytorium>/aml-lab-01/outputs

# Skopiuj pliki
cp ~/aml-lab-01-results/run_download/artifacts/outputs/model.joblib ~/<nazwa-twojego-repozytorium>/aml-lab-01/outputs/ 
cp ~/aml-lab-01-results/run_download/artifacts/outputs/metrics.json ~/<nazwa-twojego-repozytorium>/aml-lab-01/outputs/

ls -l ~/<nazwa-twojego-repozytorium>/aml-lab-01/outputs
```

Jeśli `aml-lab-01` jest w Twoim repozytorium:

```bash
git add .gitignore
git add aml-lab-01/outputs/*
git commit -m "AML Lab 01: dodano artefakty modelu i metryki do outputs/"
git push
```

---

## 🏷️ KROK 7c — Rejestracja (publikacja) modelu w AML <a id="krok7c"></a>

```bash
MODEL_NAME=sklearn-intro-<indeks>

az ml model create \
  --name $MODEL_NAME \
  --type custom_model \
  --path aml-lab-01/outputs \
  --description "Intro LogisticRegression (joblib) — AML Lab 01" \
  --tags lab=aml01 framework=scikit-learn
```

Sprawdź rejestr modeli:

```bash
az ml model list --name $MODEL_NAME -o table
```

---

## 👀 KROK 7d — Weryfikacja modelu w AML Studio <a id="krok7d"></a>

* W **Azure ML Studio → Assets → Models** znajdziesz `sklearn-intro-<indeks>`.
* W zakładce **Artifacts** zobacz `model.joblib`, a w **Details** — opis i tagi.

---

## 🧹 KROK 8 — Sprzątanie zasobów <a id="krok8"></a>

Zanim posprzątasz zasoby. Przejrzyj jakie opcje i zasoby są dostępne w Azure Machine Learning. Zwróć uwagę na Automated ML.

**Compute:**

```bash
az ml compute delete --name cpu-cluster-<indeks> --yes
```

**Infrastruktura AML (wejdź do katalogu TF):**

```bash
cd ~/<nazwa-twojego-repozytorium>/terraform/terraform-aml-lab
terraform destroy -auto-approve
```

---

## 📁 KROK 9 — Dokumentacja i commit do repo <a id="krok9"></a>

Struktura repozytorium:

```
<repo>/
├── .gitignore
├── aml-lab-01/
│   ├── job.yml
│   ├── src/
│   │   └── train.py
│   ├── outputs/
│   │   ├── model.joblib
│   │   └── metrics.json
│   └── README.md
└── terraform/
    ├── terraform-aml-lab/
    │   └── main.tf
    └── terraform-vm-lab/
        └── main.tf
```

Propozycja aml-lab-01/README.md:
```markdown
# AML Lab 01 — Pierwszy eksperyment (CLI v2)

## Uruchomienie
az extension add -n ml -y
az configure --defaults group=rg-aml-<indeks> workspace=amlws-<indeks> location=francecentral
az ml compute create --name cpu-cluster-<indeks> --type amlcompute --size Standard_DS2_v2 --min-instances 0 --max-instances 1 --idle-time-before-scale-down 120
az ml job create --file job.yml --stream

## Wyniki
- Metryki: accuracy (Studio → Experiments → intro-sklearn)
- Artefakty: `outputs/model.joblib`, `outputs/metrics.json`
```
**Commit zmian:**

```bash
git add aml-lab-01/
git commit -m "Dodanie README.md"
git push
```

---

## 📚 Dokumentacja (Microsoft/GitHub) <a id="docs"></a>

* **Azure ML CLI v2 — trenowanie/uruchamianie jobów**
  [https://learn.microsoft.com/azure/machine-learning/how-to-train-cli](https://learn.microsoft.com/azure/machine-learning/how-to-train-cli)
* **`az ml job` (CLI reference)**
  [https://learn.microsoft.com/cli/azure/ml/job](https://learn.microsoft.com/cli/azure/ml/job)
* **Rejestrowanie modeli (CLI v2)**
  [https://learn.microsoft.com/azure/machine-learning/how-to-manage-models-cli](https://learn.microsoft.com/azure/machine-learning/how-to-manage-models-cli)
* **Terraform `azurerm` (Workspace, KV, ACR, AI, LA)**
  [https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
* **GitHub PAT**
  [https://docs.github.com/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token](https://docs.github.com/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
