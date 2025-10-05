# 🧪 Instrukcja do laboratorium UMwC 2
---

## 🎯 Cel zajęć

Po wykonaniu ćwiczenia student:

* tworzy i uruchamia pełny pipeline CI dla Pythona w GitHub Actions,
* rozumie triggery (`push`, `pull_request`, `workflow_dispatch`),
* wykorzystuje **Variables**, **Secrets** i **Environments** w CI,
* generuje i publikuje **artefakt modelu ML**,
* rozumie znaczenie pliku `__init__.py` i wersjonowania zależności,
* potrafi dokumentować workflow w pliku **README.md**.

---

## 🧱 KROK 1 — Struktura repozytorium i przygotowanie plików

**Co robisz:**

1. Upewnij się, że Twoje repozytorium zawiera plik **`README.md`** (może być prosty, np. z nazwą projektu).
2. Utwórz strukturę katalogów jak poniżej.
3. W katalogu `outputs/` dodaj plik `.gitkeep` (może być pusty) – Git nie przechowuje pustych katalogów.

```
.
├─ src/
│  ├─ __init__.py
│  ├─ data.py
│  └─ model.py
├─ tests/
│  ├─ test_model_shape.py
│  └─ test_model_training.py
├─ outputs/
│  └─ .gitkeep
├─ requirements.txt
├─ requirements-dev.txt
├─ README.md
└─ .github/
   └─ workflows/
      └─ ci-ml.yml
```

---

### 🔍 Dlaczego plik `__init__.py` jest ważny?

`__init__.py` oznacza, że katalog (`src/`) jest **pakietem Pythonowym**.
Dzięki temu możesz używać importów takich jak:

```python
from src.data import get_data
from src.model import train_model
```

Bez tego pliku Python nie rozpozna katalogu jako modułu i testy zgłoszą błąd
`ModuleNotFoundError: No module named 'src'`.

> ✅ Nawet jeśli plik jest pusty, musi istnieć i być w repozytorium (commit + push).

---

## 🧠 KROK 2 — Kod Pythona (prosty model ML)

**`src/data.py`**

```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

def get_data(test_size: float = 0.2, random_state: int = 42):
    """Ładuje zbiór Iris i dzieli dane na train/test."""
    X, y = load_iris(return_X_y=True)
    return train_test_split(X, y, test_size=test_size, random_state=random_state)
```

**`src/model.py`**

```python
from sklearn.linear_model import LogisticRegression

def train_model(X_train, y_train):
    """Trenuje prosty model klasyfikacji."""
    clf = LogisticRegression(max_iter=200)
    clf.fit(X_train, y_train)
    return clf
```

---

## 🔬 KROK 3 — Testy (pytest)

**`tests/test_model_shape.py`**

```python
from src.data import get_data
from src.model import train_model

def test_predict_shape():
    X_train, X_test, y_train, y_test = get_data()
    model = train_model(X_train, y_train)
    preds = model.predict(X_test)
    assert preds.shape[0] == y_test.shape[0]
```

**`tests/test_model_training.py`**

```python
from src.data import get_data
from src.model import train_model
from sklearn.metrics import accuracy_score

def test_accuracy_minimum():
    X_train, X_test, y_train, y_test = get_data()
    model = train_model(X_train, y_train)
    acc = accuracy_score(y_test, model.predict(X_test))
    assert acc >= 0.7, f"Oczekiwane minimum accuracy 0.7, uzyskano {acc:.3f}"
```

---

## ⚙️ KROK 4 — Pliki z zależnościami (z wersjami)

**`requirements.txt`**

```
pytest==8.3.3
scikit-learn==1.5.1
joblib==1.4.2
```

**`requirements-dev.txt`**

```
black==24.8.0
flake8==7.1.1
```

💡 Dobre praktyki:

* zawsze podawaj **konkretne wersje**, by zachować powtarzalność środowiska,
* można je później „zamrozić” (`pip freeze > requirements.txt`).

---

## 🔐 KROK 5 — Ustaw zmienne i sekret w repozytorium

**Settings → Secrets and variables → Actions → Variables → New repository variable**

* `APP_NAME = IrisTrainer`
* `MODEL_STORAGE = outputs`

**Settings → Secrets and variables → Actions → Secrets → New repository secret**

* `API_TOKEN = dowolny_tekst`

---

## 🧩 KROK 6 — Workflow CI (lint + testy + model + artefakt + fallback)

**`.github/workflows/ci-ml.yml`**

```yaml
name: CI-ML

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      target_env:
        description: "Środowisko docelowe (dev/prod)"
        required: true
        default: "dev"

jobs:
  ml:
    environment: ${{ github.event_name == 'workflow_dispatch' && inputs.target_env || 'dev' }}
    runs-on: ubuntu-latest

    env:
      APP_NAME:        ${{ vars.APP_NAME }}
      MODEL_STORAGE:   ${{ vars.MODEL_STORAGE }}
      PYTHON_VERSION:  "3.11"
      DATASET_NAME:    "iris"
      API_TOKEN:       ${{ secrets.API_TOKEN }}
      TARGET_ENV:      ${{ github.event_name == 'workflow_dispatch' && inputs.target_env || 'dev' }}

    steps:
      - uses: actions/checkout@v4

      - name: Debug env
        run: |
          echo "TARGET_ENV: $TARGET_ENV"
          echo "APP_NAME: $APP_NAME"
          echo "MODEL_STORAGE: $MODEL_STORAGE"
          echo "Secret length (masked): ${#API_TOKEN}"

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install -U pip
          pip install -r requirements.txt -r requirements-dev.txt

      - name: Lint (flake8 - soft)
        run: flake8 --exit-zero src tests

      - name: Format check (black - soft)
        run: black --check src tests || true

      - name: Ensure PYTHONPATH includes repo root
        run: echo "PYTHONPATH=${PYTHONPATH}:$GITHUB_WORKSPACE" >> $GITHUB_ENV

      - name: Run tests
        run: PYTHONPATH=$PYTHONPATH:. python -m pytest -q

      - name: Train and save model
        run: |
          python - << 'PY'
          import os
          from joblib import dump
          from src.data import get_data
          from src.model import train_model

          storage = os.getenv('MODEL_STORAGE', 'outputs')
          os.makedirs(storage, exist_ok=True)

          env = os.getenv('TARGET_ENV', 'dev')
          app = os.getenv('APP_NAME', 'app')

          X_train, X_test, y_train, y_test = get_data()
          model = train_model(X_train, y_train)
          path = os.path.join(storage, f"model_{app}_{env}.joblib")
          dump(model, path)
          print(f"Model saved to {path}")
          PY

      - name: Upload model artifact
        uses: actions/upload-artifact@v4
        with:
          name: model-${{ env.TARGET_ENV }}
          path: ${{ env.MODEL_STORAGE }}/model_${{ env.APP_NAME }}_${{ env.TARGET_ENV }}.joblib
```

---

## 🧱 KROK 7 — Environments `dev` i `prod`

**Settings → Environments → New environment**

* `dev`
* `prod`

---

## ▶️ KROK 8 — Uruchom workflow

* **Ręcznie:** Actions → *CI-ML* → Run workflow → `target_env = dev` lub `prod`.
* **Automatycznie:** na push/PR na `main`.
* **Sprawdź:** logi (`Debug env`) i artefakty (`model-dev` lub `model-prod`).

---

## 🪶 KROK 9 — Uzupełnij README po zakończeniu

Dodaj do pliku **`README.md`** sekcję opisującą workflow:

```markdown
## 🧩 Continuous Integration – GitHub Actions

Ten projekt zawiera workflow **CI-ML**, który:
- uruchamia się automatycznie na push, PR lub manualnie,
- instaluje zależności z plików `requirements*.txt`,
- wykonuje lint (flake8) i format check (black),
- uruchamia testy pytest,
- trenuje model ML (Logistic Regression),
- publikuje model jako artefakt z nazwą środowiska (`model-dev`, `model-prod`),
- wykorzystuje repozytoryjne Variables i Secrets.
```

---

## 💡 Zadanie dodatkowe – sterowanie parametrami przez zmienne środowiskowe

**Cel:** pokazać, jak zmieniać zachowanie kodu bez edycji jego treści.

1. W pliku `src/model.py` zmodyfikuj funkcję:

   ```python
   import os
   from sklearn.linear_model import LogisticRegression

   def train_model(X_train, y_train):
       """Trenuje model z konfigurowalnym max_iter."""
       max_iter = int(os.getenv("MAX_ITER", 200))
       print(f"Training LogisticRegression(max_iter={max_iter})")
       clf = LogisticRegression(max_iter=max_iter)
       clf.fit(X_train, y_train)
       return clf
   ```
2. W **Settings → Variables** dodaj nową zmienną:

   ```
   MAX_ITER = 300
   ```
3. Uruchom workflow — w logach zobaczysz:

   ```
   Training LogisticRegression(max_iter=300)
   ```


---

## 📚 Dokumentacja i źródła

* [GitHub Actions: workflow syntax](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions)
* [GitHub Actions: Variables & Secrets](https://docs.github.com/actions/learn-github-actions/variables)
* [Azure Machine Learning Python SDK](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-ml-readme?view=azure-python)
* [flake8 docs](https://flake8.pycqa.org)
* [pytest docs](https://docs.pytest.org)
