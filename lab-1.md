# ✅ Instrukcja do laboratorium UMwC 1

---

## 🧾 Cel zajęć

Celem dzisiejszych zajęć jest:

* Założenie konta na platformie GitHub.
* Utworzenie pierwszego repozytorium.
* Dodanie pliku `README.md` do repozytorium.
* Stworzenie prostego workflow GitHub Actions, który uruchamia się automatycznie po każdym commicie.

---

## 🔧 Krok 1 – Założenie konta na GitHub

1. Wejdź na stronę: [https://github.com/](https://github.com/)
2. Kliknij przycisk **Sign up**.
3. Podaj swój adres e-mail, nazwę użytkownika i hasło.
4. Potwierdź e-mail, na skrzynkę e-mail dostałeś kod potwierdzający.
5. Po rejestracji możesz się zalogować do swojego konta.

---

## 📁 Krok 2 – Stworzenie nowego repozytorium

1. Po zalogowaniu kliknij w **Create repository**.
2. Uzupełnij pola:
   * **Repository name**: np. `umwc-projekt-<id>`
   * **Description** (opcjonalnie): `Repozytorium projektu`
   * Zaznacz **Private**
   * Zaznacz opcję **Add a README file**
3. Kliknij **Create repository**

---

## 📝 Krok 3 – Edycja pliku `README.md`

1. W repozytorium kliknij na plik `README.md`.
2. Kliknij ikonę ołówka (Edit).
3. Zmień zawartość na coś własnego, np.:

```
   # Projekt UMwC
   To jest mój pierwszy projekt na GitHub. 🚀
```

4. Na dole strony wpisz opis zmiany w polu **Commit changes** (np. `Zmieniono README`) i kliknij **Commit changes**.

---

## 🤖 Krok 4 – Dodanie workflow GitHub Actions

1. W repozytorium kliknij zakładkę **Actions**.

2. Kliknij **set up a workflow yourself**

3. Nazwij workflow, np.: `.github/workflows/main.yml`

4. Wklej poniższy kod:

   ```yaml
   name: Pierwszy workflow

   on:
     push:
       branches: [ "main" ]
     pull_request:
       branches: [ "main" ]

   jobs:
     build:
       runs-on: ubuntu-latest

       steps:
         - name: Checkout repo
           uses: actions/checkout@v3

         - name: Wyświetl zawartość katalogu
           run: ls -la

         - name: Wyświetl wiadomość
           run: echo "Gratulacje! Twój pierwszy workflow działa! 🎉"
   ```

5. Kliknij **Commit changes...**, a potem **Commit changes**.

---

## ✅ Krok 5 – Sprawdzenie działania workflow

1. Po zapisaniu workflow, GitHub automatycznie uruchomi go.
2. Przejdź do zakładki **Actions**, gdzie zobaczysz status wykonania.
3. Kliknij na nazwę workflow, a potem na konkretną wersję (run), aby podejrzeć logi z działania.
