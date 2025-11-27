---

# 🧪 AML Lab 02 – Dataset, trening modelu i rejestracja w Azure Machine Learning



---

## 📚 Spis treści


1. [KROK 0 – Klon repozytorium (GitHub + PAT)](#k0)
2. [KROK 1 – Odtworzenie AML Workspace z Terraform](#k1)
3. [KROK 2 – Konfiguracja Azure ML CLI v2 i Compute Cluster](#k2)
4. [KROK 3 – Struktura katalogu `aml-lab-02`](#k3)
5. [KROK 4 – Job do generowania danych (`data_prep.py` + `data_job.yml`)](#k4)
6. [KROK 5 – Pobranie danych i rejestracja Data Asset](#k5)
7. [KROK 6 – Kod treningowy modelu (`train.py`)](#k6)
8. [KROK 7 – Training job (`training_job.yml`)](#k7)
9. [KROK 8 – Rejestracja modelu](#k8)
10. [KROK 9 – Sprzątanie zasobów](#k9)
11. [KROK 10 – Dodanie plików do repozytorium](#k10)
12. [Dokumentacja](#docs)

---



## 🔐 KROK 0 – Klon repozytorium (GitHub + PAT) <a id="k0"></a>

Kroki 0 i 1, są dokładnie takie jak w poprzednim ćwiczeniu. Jeśli masz zapisany swój Token, możesz przejśc od razu do klonowania repozytorium.

Pracujemy w **Azure Cloud Shell (Bash)**: [https://shell.azure.com](https://shell.azure.com)

### 0.1 Konfiguracja Gita

```bash
git config --global user.name  "Imię Nazwisko"
git config --global user.email "twoj.mail@dsw.edu.pl"
git config --global credential.helper 'cache --timeout=3600'
```

### 0.2 Personal Access Token (PAT) w GitHub

1. GitHub → **Settings → Developer settings → Personal access tokens**.
2. Utwórz **Fine-grained token** z dostępem do wybranego repozytorium (Contents: Read/Write).
3. Zapisz token – będzie użyty jako hasło.

### 0.3 Klon repozytorium w Cloud Shell

```bash
git clone https://github.com/<owner>/<repo>.git
cd <repo>
```

Podczas klonowania:

* **Username** = Twój login GitHub
* **Password** = wygenerowany token **PAT**

Upewnij się, że w repo jest `.gitignore` ignorujący pliki Terraforma, np.:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
```

---

## ⚙️ KROK 1 – Odtworzenie AML Workspace z Terraform <a id="k1"></a>

Zakładamy, że z poprzednich ćwiczeń masz w repo:

```text
terraform/
  terraform-aml-lab/
  terraform-vm-lab/
```

### 1.1 Przejście do katalogu Terraforma dla AML

```bash
cd terraform/terraform-aml-lab
```

### 1.2 Sprawdzenie / ustawienie subskrypcji

```bash
az account show --output table
az account set --subscription "Azure for Students"
```

### 1.3 Inicjalizacja i wdrożenie

```bash
terraform init
terraform plan
terraform apply -auto-approve
terraform output
```

W outputach powinny być m.in.:

* nazwa resource group, np. `rg-aml-<indeks>`,
* nazwa workspace, np. `amlws-<indeks>`,
* link do AML Studio.

---

## ☁️ KROK 2 – Konfiguracja Azure ML CLI v2 i Compute Cluster <a id="k2"></a>

### 2.1 Instalacja rozszerzenia ML

```bash
az extension add -n ml -y
```

### 2.2 Ustawienie domyślnych parametrów CLI

```bash
az configure --defaults group=rg-aml-<indeks> workspace=amlws-<indeks> location=francecentral
```

### 2.3 Utworzenie AML Compute Cluster

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



---

## 📁 KROK 3 – Struktura katalogu `aml-lab-02` <a id="k3"></a>

Wracamy do głównego katalogu repozytorium:

```bash
cd ~/<nazwa-twojego-repozytorium>
mkdir -p aml-lab-02/src
cd aml-lab-02
```

Docelowo w `aml-lab-02/` będziemy mieli:

```text
aml-lab-02/
├── data_job.yml
├── training_job.yml
└── src/
    ├── data_prep.py
    └── train.py
```

---

## 💾 KROK 4 – Job do generowania danych (data_prep.py + data_job.yml) <a id="k4"></a>

### 4.1 Skrypt `src/data_prep.py`

Przejdź do katalogu `src`:

```bash
cd ~/<nazwa-twojego-repozytorium>/aml-lab-02/src
```

Utwórz plik `data_prep.py`:

```python
from pathlib import Path

import pandas as pd
from sklearn.datasets import make_classification


def main():
    print("Generuję dataset…")
    X, y = make_classification(
        n_samples=500,
        n_features=10,
        n_informative=4,
        random_state=24,
    )

    df = pd.DataFrame(X, columns=[f"f{i}" for i in range(10)])
    df["label"] = y

    out_dir = Path("outputs")
    out_dir.mkdir(parents=True, exist_ok=True)
    out_path = out_dir / "data.csv"

    df.to_csv(out_path, index=False)
    print(f"✔ Dataset zapisany: {out_path}")


if __name__ == "__main__":
    main()
```

> `outputs/` to domyślny katalog artefaktów w AML – wszystko, co tam zapiszemy, będzie dostępne po zakończeniu joba.

### 4.2 Definicja joba danych – `data_job.yml`

Wróć do katalogu `aml-lab-02`:

```bash
cd ~/<nazwa-twojego-repozytorium>/aml-lab-02
```

Utwórz plik `data_job.yml`:

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
type: command
display_name: aml02-data-prep-job
experiment_name: aml02-data-prep

code: .
command: python src/data_prep.py

environment: azureml://registries/azureml/environments/sklearn-1.5/versions/34
compute: azureml:cpu-cluster-<indeks>

limits:
  timeout: 3600
```

### 4.3 Uruchomienie joba generującego dane

```bash
JOB_DATA=$(az ml job create --file data_job.yml --query name -o tsv)
az ml job stream --name $JOB_DATA
```

Po zakończeniu joba w AML Studio (Jobs → `aml02-data-prep`) w zakładce *Outputs + logs* powinien być plik `outputs/data.csv`.

---

## 📦 KROK 5 – Pobranie danych i rejestracja Data Asset <a id="k5"></a>

### 5.1 Pobierz artefakty joba do Cloud Shell

```bash
az ml job download --name $JOB_DATA --download-path downloaded_data
ls -R downloaded_data
# powinno być: downloaded_data/artifacts/outputs/data.csv
```

### 5.2 Rejestracja Data Asset z lokalnego pliku

```bash
az ml data create \
  --name demo-data-<indeks> \
  --path downloaded_data/outputs/data.csv \
  --type uri_file \
  --description "Dataset wygenerowany w AML Lab 02"
```

Sprawdź:

```bash
az ml data list -o table
```

---

## 🧠 KROK 6 – Kod treningowy modelu (`train.py`) <a id="k6"></a>

Teraz napiszemy skrypt, który:

* przyjmuje ścieżkę do pliku CSV jako argument `--data`,
* trenuje prosty model logistycznej regresji,
* zapisuje wynik do `outputs/model.joblib` i `outputs/metrics.json`.

Przejdź do `src`:

```bash
cd ~/<nazwa-twojego-repozytorium>/aml-lab-02/src
```

Utwórz plik `train.py`:

```python
import argparse
import json
from pathlib import Path

import joblib
import pandas as pd
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split


def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--data",
        type=str,
        required=True,
        help="Ścieżka do pliku CSV z danymi treningowymi",
    )
    return parser.parse_args()


def main():
    args = parse_args()
    data_path = args.data
    print(f"Czytam dane z: {data_path}")

    df = pd.read_csv(data_path)

    X = df.drop("label", axis=1)
    y = df["label"]

    Xtr, Xte, ytr, yte = train_test_split(
        X, y, test_size=0.3, random_state=42
    )

    clf = LogisticRegression(max_iter=1000)
    clf.fit(Xtr, ytr)

    acc = accuracy_score(yte, clf.predict(Xte))
    print(f"Dokładność modelu: {acc:.4f}")

    out_dir = Path("outputs")
    out_dir.mkdir(parents=True, exist_ok=True)

    (out_dir / "metrics.json").write_text(
        json.dumps({"accuracy": acc}, indent=2)
    )
    joblib.dump(clf, out_dir / "model.joblib")

    print("✔ Model i metryki zapisane w katalogu outputs/")


if __name__ == "__main__":
    main()
```



---

## ▶️ KROK 7 – Training job (`training_job.yml`) <a id="k7"></a>

Wróć do `aml-lab-02`:

```bash
cd ~/<nazwa-twojego-repozytorium>/aml-lab-02
```

Utwórz plik `training_job.yml`:

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
type: command

display_name: aml02-sklearn-training-job
experiment_name: aml02-training

code: .
command: >
  python src/train.py
  --data ${{inputs.training_data}}

environment: azureml://registries/azureml/environments/sklearn-1.5/versions/34
compute: azureml:cpu-cluster-<indeks>

inputs:
  training_data:
    type: uri_file
    path: azureml:demo-data-<indeks>@latest

limits:
  timeout: 3600
```

Co tu się dzieje:

* `inputs.training_data` wskazuje na **Data Asset** `demo-data-<indeks>`,
* AML mapuje asset na lokalną ścieżkę w kontenerze i wstrzykuje ją w miejsce `${{inputs.training_data}}`,
* ostatecznie w kontenerze działa komenda np.:
  `python src/train.py --data /mnt/azureml/.../data.csv`,
* `train.py` ładuje dane z tej ścieżki i zapisuje wyniki do `outputs/`.

### Uruchomienie training job

```bash
JOB_TRAIN=$(az ml job create --file training_job.yml --query name -o tsv)
az ml job stream --name $JOB_TRAIN
```

Po zakończeniu:

* w Azure ML Studio → *Jobs → aml02-training → Run*
  możesz sprawdzić:

  * zakładkę **Metrics** (accuracy),
  * zakładkę **Outputs + logs** (`outputs/model.joblib`, `outputs/metrics.json`).

---

## 🏷️ KROK 8 – Rejestracja modelu <a id="k8"></a>

Najpierw pobierz artefakty lokalnie:

```bash
mkdir -p ~/aml-lab-02-results/train_run_download
az ml job download --name $JOB_TRAIN --download-path ~/aml-lab-02-results/train_run_download
ls -R ~/aml-lab-02-results/train_run_download
# oczekiwane: .../artifacts/outputs/model.joblib i metrics.json
```

Załóżmy, że model leży w:

```text
~/aml-lab-02-results/train_run_download/artifacts/outputs/
```

Rejestracja modelu:

```bash
MODEL_NAME=sklearn-intro-<indeks>

az ml model create \
  --name $MODEL_NAME \
  --type custom_model \
  --path ~/aml-lab-02-results/train_run_download/artifacts/outputs \
  --description "Model LogisticRegression – AML Lab 02" \
  --tags "lab=aml02 framework=sklearn"
```

Sprawdź modele:

```bash
az ml model list --name $MODEL_NAME -o table
```

W Azure ML Studio → **Assets → Models** powinieneś zobaczyć `sklearn-intro-<indeks>`.

Zrób screenshot zakładki Details dla nowo utworzonego modelu. Dodaj do repozytorium jako dokumentacja

---

## 🧹 KROK 9 – Sprzątanie zasobów <a id="k9"></a>

### 9.1 Usuń compute cluster

```bash
az ml compute delete --name cpu-cluster-<indeks> --yes
```

### 9.2 Usuń infrastrukturę AML (Terraform)

```bash
cd ~/<nazwa-twojego-repozytorium>/terraform/terraform-aml-lab
terraform destroy -auto-approve
```



---

## 📁 KROK 10 – Dodanie plików do repozytorium <a id="k10"></a>

Wracamy do root repo:

```bash
cd ~/<nazwa-twojego-repozytorium>
```

Struktura powinna wyglądać mniej więcej tak:

```text
<repo>/
├── terraform/
│   ├── terraform-aml-lab/
│   └── terraform-vm-lab/
└── aml-lab-02/
    ├── model-details.jpg
    ├── data_job.yml
    ├── training_job.yml
    └── src/
        ├── data_prep.py
        └── train.py
```

Dodaj i wypchnij zmiany:

```bash
git add aml-lab-02
git commit -m "AML Lab 02 – data asset + training job + model registry"
git push
```

---


