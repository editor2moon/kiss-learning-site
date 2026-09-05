# Git Cheatsheet KISS

> Cel: szybko sprawdzić stan repozytorium, odzyskać pliki, synchronizować branche, przenosić commity i bezpiecznie rozwiązywać konflikty.

## Zasada bezpieczeństwa

Przed operacją zmieniającą historię lub usuwającą pliki wykonaj:

```bash
git status
git branch --show-current
git log --oneline --graph --decorate -10
```

Jeżeli chcesz dodatkowo zabezpieczyć bieżące zmiany:

```bash
git stash push -u -m "backup przed operacja"
```

> Uwaga: `reset --hard`, `clean -fd` i wymuszony push mogą usunąć lokalną pracę.

---

## 1. Sprawdzenie stanu repozytorium

```bash
git status
```

Pokazuje:

- aktualny branch,
- zmienione i usunięte pliki,
- pliki dodane do staging area,
- pliki nieśledzone,
- informację, czy branch jest przed lub za branchem zdalnym.

Krótka wersja:

```bash
git status -sb
```

---

## 2. Historia commitów

```bash
git log --oneline
```

Historia ze strukturą branchy:

```bash
git log --oneline --graph --decorate --all
```

Ostatnich 10 commitów:

```bash
git log --oneline -10
```

---

## 3. Sprawdzenie różnic

Zmiany w plikach, których jeszcze nie dodano przez `git add`:

```bash
git diff
```

Zmiany dodane do staging area:

```bash
git diff --staged
```

Różnice między lokalnym branchem a GitHubem:

```bash
git fetch origin
git diff HEAD..origin/main
```

Commity dostępne na GitHubie, których nie masz lokalnie:

```bash
git log --oneline HEAD..origin/main
```

Lokalne commity, których nie ma na GitHubie:

```bash
git log --oneline origin/main..HEAD
```

---

## 4. Fetch i pull

### Fetch

```bash
git fetch origin
```

Mental model:

```text
Pobierz informacje z GitHuba, ale nie zmieniaj moich plików.
```

Aktualizuje między innymi referencje:

```text
origin/main
origin/develop
```

### Pull

```bash
git pull
```

Najprościej:

```text
git pull = git fetch + integracja zmian
```

W zależności od konfiguracji integracją może być merge albo rebase.

Jawny pull z merge:

```bash
git pull --no-rebase
```

Jawny pull z rebase:

```bash
git pull --rebase
```

---

## 5. Przywrócenie lokalnie usuniętych lub zmienionych plików

### Jeden plik

```bash
git restore path/to/file.txt
```

### Wszystkie śledzone pliki

```bash
git restore .
```

Przywraca zawartość z indexu. Jeżeli pliku nie dodano wcześniej do staging area, zwykle oznacza to stan z ostatniego commita.

### Plik dodany do staging area

Najpierw usuń go ze staging area, potem przywróć:

```bash
git restore --staged path/to/file.txt
git restore path/to/file.txt
```

### Starsza składnia

```bash
git checkout HEAD -- path/to/file.txt
```

---

## 6. Wyrzucenie wszystkich zmian w śledzonych plikach

```bash
git reset --hard HEAD
```

Efekt:

- branch pozostaje na ostatnim lokalnym commicie,
- staging area wraca do tego commita,
- śledzone pliki wracają do tego commita.

> Nie usuwa nieśledzonych plików. Do tego służy `git clean`.

---

## 7. Ustawienie repozytorium dokładnie jak na GitHubie

Najpierw sprawdź nazwę brancha:

```bash
git branch --show-current
```

Pobierz aktualny stan zdalny:

```bash
git fetch origin
```

Ustaw lokalny branch jak `origin/main`:

```bash
git reset --hard origin/main
```

Mental model:

```text
Lokalny branch i śledzone pliki = origin/main
```

Co robi ta operacja:

1. `fetch` pobiera informacje o commitach i branchach.
2. `reset --hard` przesuwa lokalny branch na commit wskazywany przez `origin/main`.
3. Resetuje staging area.
4. Nadpisuje śledzone pliki stanem z `origin/main`.
5. Usuwa lokalne, niepushowane zmiany w śledzonych plikach.

Dla innego brancha:

```bash
git fetch origin
git reset --hard origin/develop
```

> Uwaga: najpierw upewnij się, że resetujesz do poprawnego brancha zdalnego.

---

## 8. Nieśledzone pliki i katalogi

Podgląd tego, co zostałoby usunięte:

```bash
git clean -nd
```

Usuń nieśledzone pliki i katalogi:

```bash
git clean -fd
```

Podgląd również plików ignorowanych przez `.gitignore`:

```bash
git clean -ndx
```

Usuń także ignorowane pliki:

```bash
git clean -fdx
```

> `-fdx` jest bardzo destrukcyjne. Może usunąć lokalne pliki `.env`, buildy i cache.

---

## 9. Emergency reset do stanu GitHuba

```bash
git fetch origin
git reset --hard origin/main
git clean -nd
git clean -fd
```

KISS:

```text
fetch        -> sprawdź najnowszy stan GitHuba
reset --hard -> ustaw śledzone pliki i branch jak na GitHubie
clean -nd    -> pokaż nieśledzone pliki do usunięcia
clean -fd    -> usuń nieśledzone pliki i katalogi
```

---

## 10. Cofnięcie ostatniego commita

### Usuń commit, ale zostaw zmiany w staging area

```bash
git reset --soft HEAD~1
```

### Usuń commit i wycofaj zmiany ze staging area, ale zachowaj pliki

```bash
git reset --mixed HEAD~1
```

Skrót, ponieważ `--mixed` jest domyślne:

```bash
git reset HEAD~1
```

### Usuń commit razem ze zmianami w plikach

```bash
git reset --hard HEAD~1
```

### Bezpieczne cofnięcie commita, który został już wypchnięty

```bash
git revert <commit_id>
```

`revert` tworzy nowy commit odwracający wcześniejsze zmiany. Jest bezpieczniejszy dla współdzielonej historii.

---

## 11. Stash

Schowaj zmiany w śledzonych plikach:

```bash
git stash push -m "opis zmian"
```

Schowaj również nieśledzone pliki:

```bash
git stash push -u -m "opis zmian"
```

Lista stashy:

```bash
git stash list
```

Podejrzyj zmiany:

```bash
git stash show -p stash@{0}
```

Przywróć i usuń stash z listy:

```bash
git stash pop
```

Przywróć bez usuwania stashu:

```bash
git stash apply stash@{0}
```

Usuń jeden stash:

```bash
git stash drop stash@{0}
```

---

## 12. Branche

Lista lokalnych branchy:

```bash
git branch
```

Lista lokalnych i zdalnych branchy:

```bash
git branch -a
```

Przełącz branch:

```bash
git switch develop
```

Utwórz nowy branch i przełącz się na niego:

```bash
git switch -c feature/new-feature
```

Usuń scalony branch lokalny:

```bash
git branch -d feature/new-feature
```

Wymuś usunięcie niescalonego brancha:

```bash
git branch -D feature/new-feature
```

Wyślij nowy branch i ustaw tracking:

```bash
git push -u origin feature/new-feature
```

---

## 13. Cherry-pick

Cherry-pick kopiuje wybrany commit na aktualny branch.

### Przykład

Przejdź na branch docelowy:

```bash
git switch main
```

Pobierz aktualne informacje:

```bash
git fetch origin
```

Skopiuj commit:

```bash
git cherry-pick abc1234
```

### Kilka commitów

Wybrane commity:

```bash
git cherry-pick abc1234 def5678
```

Zakres commitów, bez pierwszego wskazanego commita:

```bash
git cherry-pick abc1234..def5678
```

Zakres razem z pierwszym commitem:

```bash
git cherry-pick abc1234^..def5678
```

### Konflikt podczas cherry-pick

```bash
git status
```

Napraw pliki, a następnie:

```bash
git add <naprawiony_plik>
git cherry-pick --continue
```

Przerwanie operacji:

```bash
git cherry-pick --abort
```

Pominięcie problematycznego commita:

```bash
git cherry-pick --skip
```

---

## 14. Merge

Merge łączy historię jednego brancha z aktualnym branchem.

### Przykład: feature do main

```bash
git switch main
git fetch origin
git pull --ff-only
git merge feature/new-feature
```

Mental model:

```text
Jestem na branchu docelowym i dołączam branch źródłowy.
```

Historia przed merge:

```text
A---B---C  main
     \
      D---E  feature
```

Możliwa historia po merge:

```text
A---B---C-------M  main
     \         /
      D-------E  feature
```

### Fast-forward

Jeśli na branchu docelowym nie ma nowych commitów, Git może tylko przesunąć wskaźnik:

```bash
git merge --ff-only feature/new-feature
```

Komenda zakończy się błędem, jeśli fast-forward nie jest możliwy.

### Zawsze utwórz merge commit

```bash
git merge --no-ff feature/new-feature
```

### Przerwij merge z konfliktem

```bash
git merge --abort
```

### Kiedy używać merge

- gdy branch jest współdzielony,
- gdy chcesz zachować rzeczywistą strukturę historii,
- gdy polityka zespołu wymaga merge commitów.

---

## 15. Rebase

Rebase przenosi własne commity na nową bazę i tworzy dla nich nowe identyfikatory.

### Aktualizacja feature brancha względem `origin/main`

```bash
git switch feature/new-feature
git fetch origin
git rebase origin/main
```

Przed:

```text
A---B---C---F---G  main
     \
      D---E  feature
```

Po:

```text
A---B---C---F---G---D'---E'  feature
```

`D'` i `E'` zawierają podobne zmiany, ale są nowymi commitami.

### Konflikt podczas rebase

```bash
git status
```

Napraw pliki i wykonaj:

```bash
git add <naprawiony_plik>
git rebase --continue
```

Pomiń bieżący commit:

```bash
git rebase --skip
```

Przerwij rebase i wróć do stanu początkowego:

```bash
git rebase --abort
```

### Push po rebase

Jeżeli branch był wcześniej wypchnięty i tylko Ty nad nim pracujesz:

```bash
git push --force-with-lease
```

> Preferuj `--force-with-lease` zamiast `--force`. Nie wykonuj rebase współdzielonej historii bez uzgodnienia z zespołem.

---

## 16. Merge vs rebase

| Cecha | Merge | Rebase |
|---|---|---|
| Łączy historie | Tak | Tak |
| Przepisuje istniejące commity | Nie | Tak |
| Może utworzyć merge commit | Tak | Nie |
| Zachowuje strukturę branchy | Tak | Nie w tej samej formie |
| Dobra opcja dla współdzielonego brancha | Zwykle tak | Zwykle nie |
| Dobra opcja dla prywatnego feature brancha | Tak | Tak |
| Może wymagać force push | Nie | Tak, jeśli branch był już wypchnięty |

Reguła KISS:

```text
Współdzielony branch -> merge
Prywatny feature branch -> rebase może uporządkować historię
Polityka projektu -> zawsze ma pierwszeństwo
```

---

## 17. Rozwiązywanie konfliktów

### Jak wygląda konflikt w pliku

```text
<<<<<<< HEAD
wersja z aktualnego brancha
=======
wersja z dołączanego commita lub brancha
>>>>>>> feature/new-feature
```

Znaczenie:

- `<<<<<<< HEAD` rozpoczyna wersję aktualnie checkoutowaną,
- `=======` oddziela wersje,
- `>>>>>>> ...` kończy drugą wersję.

### Uniwersalny workflow KISS

1. Sprawdź stan:

```bash
git status
```

2. Otwórz każdy plik oznaczony jako konfliktowy.
3. Wybierz poprawną wersję albo połącz obie wersje.
4. Usuń znaczniki konfliktu.
5. Sprawdź, czy znaczniki nie pozostały:

```bash
git grep -n -e '^<<<<<<<' -e '^=======' -e '^>>>>>>>'
```

6. Uruchom testy lub walidację projektu.
7. Oznacz plik jako rozwiązany:

```bash
git add <naprawiony_plik>
```

8. Dokończ właściwą operację:

```bash
# Merge
git commit

# Rebase
git rebase --continue

# Cherry-pick
git cherry-pick --continue
```

### Rezygnacja z operacji

```bash
# Merge
git merge --abort

# Rebase
git rebase --abort

# Cherry-pick
git cherry-pick --abort
```

### Użycie jednej wersji całego pliku podczas merge

Zachowaj wersję aktualnego brancha:

```bash
git checkout --ours path/to/file
```

Zachowaj wersję dołączanego brancha:

```bash
git checkout --theirs path/to/file
```

Następnie:

```bash
git add path/to/file
```

> Uwaga: podczas rebase znaczenie `ours` i `theirs` może być nieintuicyjne. Zawsze sprawdź wynik przez `git diff`.

---

## 18. Reflog, czyli komenda ratunkowa

```bash
git reflog
```

Pokazuje wcześniejsze pozycje `HEAD`, również po resetach i rebase.

Utwórz branch ratunkowy ze znalezionego punktu:

```bash
git branch rescue-branch HEAD@{2}
```

Albo przywróć repo do wybranego punktu:

```bash
git reset --hard HEAD@{2}
```

Bezpieczniej najpierw utworzyć branch ratunkowy, a dopiero potem wykonywać reset.

---

## 19. Najważniejsze komendy awaryjne

| Problem | Komenda |
|---|---|
| Nie wiem, co się dzieje | `git status` |
| Chcę zobaczyć historię | `git log --oneline --graph --decorate --all` |
| Chcę zobaczyć wykonane operacje | `git reflog` |
| Chcę przerwać merge | `git merge --abort` |
| Chcę przerwać rebase | `git rebase --abort` |
| Chcę przerwać cherry-pick | `git cherry-pick --abort` |
| Chcę odzyskać lokalnie usunięty plik | `git restore <plik>` |
| Chcę wyrzucić zmiany w śledzonych plikach | `git reset --hard HEAD` |
| Chcę stan śledzonych plików jak na GitHubie | `git fetch origin && git reset --hard origin/main` |
| Chcę podejrzeć nieśledzone pliki do usunięcia | `git clean -nd` |

---

## 20. DevOps workflow przed rozpoczęciem pracy

```bash
git switch main
git fetch origin
git pull --ff-only
git switch -c feature/nazwa-zmiany
```

Praca i commit:

```bash
git status
git diff
git add <pliki>
git diff --staged
git commit -m "Opis zmiany"
git push -u origin feature/nazwa-zmiany
```

Aktualizacja feature brancha przez rebase:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```

Alternatywnie aktualizacja przez merge:

```bash
git fetch origin
git merge origin/main
git push
```

---

## 21. Git mental model KISS

```text
status        -> Co się dzieje?
log           -> Co zostało zapisane?
diff          -> Co się zmieniło?
fetch         -> Co jest na serwerze?
pull          -> Pobierz i zintegruj zmiany.
restore       -> Przywróć zawartość pliku.
stash         -> Tymczasowo schowaj zmiany.
merge         -> Połącz historie branchy.
rebase        -> Przenieś commity na nową bazę.
cherry-pick   -> Skopiuj wybrany commit.
reset         -> Przesuń branch i opcjonalnie przywróć pliki.
revert        -> Odwróć commit nowym commitem.
clean         -> Usuń nieśledzone pliki.
reflog        -> Znajdź wcześniejsze pozycje HEAD.
```

---

## Złota zasada

```bash
git status
git diff
git log --oneline --graph --decorate -10
```

Dopiero potem wykonuj operacje destrukcyjne.

Jeżeli coś poszło nie tak:

```bash
git reflog
```
