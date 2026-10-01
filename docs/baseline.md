# Baseline — pomiar stanu wyjściowego

Plik uzupełniasz w ZADANIU 01. Czasy odczytujesz w zakładce **Actions**. Wystarczy dokładność
do sekundy.

## Czasy kroków — przebieg na `main`

| Krok | Czas |
|---|---|
| Set up job | 2s |
| Checkout | 1s |
| Set up Node | 14s |
| Install dependencies | 10s |
| Install Playwright browsers | 19s |
| Unit tests | 1s |
| API tests | 20s |
| UI tests | 2m 29s |
| Upload Playwright report | 2s |
| Post Set up Node | 0s |
| Post Checkout | 1s |
| Complete job | 0s |
| **Cały przebieg** | ok. 3m 39s (suma kroków) |

## Czas do pierwszego czerwonego sygnału

| Branch | Czas samego testu | Od startu przebiegu do informacji o błędzie |
|---|---|---|
| `demo/failing-unit` | 1s | ok. 28s (suma kroków do Unit tests: 1+1+0+6+19+1) |
| `demo/failing-search` | — | ok. 4m 7s (UI tests 3m 15s, czerwony dopiero na końcu kroku) |

Na `demo/failing-unit` kroki **API tests**, **UI tests**, **Upload Playwright report** i **Post Set up Node** zostały
pominięte (skipped), bo krok Unit tests padł, a kolejne kroki domyślnie nie wykonują się po błędzie. Dzięki temu
przebieg skończył się po ok. 36s zamiast ok. 3m 39s, ale kosztem tego, że nie wiemy, czy API i UI też są zepsute.

## `demo/failing-lint` i `demo/failing-security`

| Branch | Wynik przebiegu | Dlaczego tak |
|---|---|---|
| `demo/failing-lint` | zielony (success), ok. 3m 22s | Pipeline nie ma kroku lintu, więc reguła złamana w koszyku nie jest nigdzie sprawdzana. Testy przechodzą, bo kod działa, tylko jest niezgodny ze stylem. |
| `demo/failing-security` | zielony (success), ok. 3m 33s | Pipeline nie ma skanowania sekretów ani zależności, więc token API wpisany w kod płatności przechodzi bez ostrzeżenia. Testy nie sprawdzają bezpieczeństwa. |

Zielony przebieg oznacza tylko, że przeszły testy, które mamy. Nie znaczy, że zmiana jest bezpieczna ani zgodna ze standardami.

## Pięć problemów obecnego pipeline’u

1. **Wszystko jest w jednym jobie, kroki idą po kolei.** Testy jednostkowe (1s), API (ok. 20s) i UI (ok. 2,5 min) nie biegną równolegle, więc całość trwa ok. 3m 39s.
2. **Późny feedback.** Najszybszy test (unit, 1s) jest dopiero po instalacji przeglądarek Playwright (ok. 19s). Błąd z UI widać dopiero po ok. 4 minutach (`demo/failing-search`).
3. **Brak lintu i skanowania bezpieczeństwa.** `demo/failing-lint` i `demo/failing-security` przechodzą na zielono, mimo złamanej reguły i tokena w kodzie.
4. **Brak cache.** `npm ci` i `npx playwright install` pobierają wszystko od zera przy każdym pushu (ok. 10s + 19s), a przeglądarki instalowane są wszystkie (`npx playwright install` bez wskazania jednej).
5. **Marnowanie zasobów i brak kontroli uruchomień.** Trigger `on: push` uruchamia pełny pipeline dla każdego brancha i każdego commita, bez `concurrency` (stare przebiegi nie są anulowane), bez `paths` (zmiana samej dokumentacji też uruchamia ok. 4 minuty testów) i bez `timeout-minutes`. Dodatkowo raport Playwright jest wgrywany tylko po sukcesie (krok jest pomijany po błędzie), czyli wtedy, gdy raport jest najbardziej potrzebny.

## Pomiary z kolejnych zadań

Tu dopisujesz pomiary i odpowiedzi z kolejnych zadań, pod nagłówkiem z numerem zadania.

### ZADANIE 03 — Szybkie kontrole najpierw i cache

| Pomiar | Przed (`main`) | Po: pierwszy przebieg (cache pusty) | Po: drugi przebieg (re-run, cache trafiony) |
|---|---|---|---|
| Install dependencies | 10s | 5s | 6s |
| Install Playwright browsers | 19s | 10s | pominięty (0s) |
| Cały przebieg | ok. 3m 39s | 3m 18s | 3m 5s |
| Czas do informacji o błędzie lintu | nigdy | | |

W drugim przebiegu log kroku Cache Playwright browsers pokazuje `Cache restored from key: playwright-Linux-1.62.1`,
a krok Install Playwright browsers jest pominięty. Krok Cache Playwright browsers trwa 4s (pobranie ok. 269 MB),
więc oszczędność jest mała. Większość czasu to nadal UI tests (ok. 2m 20s).
