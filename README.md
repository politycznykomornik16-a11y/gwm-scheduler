# gwm-scheduler

Publiczny harmonogram collectora **Galactic War Monitor** (HELLDIVERS 2).
To repo zawiera **wyłącznie** workflow GitHub Actions uruchamiający co 5 minut
jednorazowy poll collectora z **prywatnego** repo aplikacji. Zero kodu aplikacji.

## Co robi workflow (`.github/workflows/collector.yml`)

1. Trigger: `schedule` co 5 minut (`*/5 * * * *`) albo ręcznie (`workflow_dispatch`).
2. Checkout **prywatnego** repo aplikacji przez **deploy key tylko-do-odczytu**.
3. `npm ci` → `npm run db:generate -w @gwm/db` → `npm run build` (komendy wg README aplikacji).
4. `npm run collector:poll` — jeden pełny cykl: community API → normalizacja → baza (stan + snapshoty).

`concurrency: group=collector, cancel-in-progress=false` — collector działa jako
**dokładnie jedna instancja naraz**: kolejne uruchomienie czeka w kolejce i nie
przerywa bieżącego polla. Drugą warstwą jest advisory lock wbudowany w collectora
(zapis pomijany, gdy inna instancja pisze).

## Sekrety (GitHub Secrets tego repo — wartości NIGDY nie trafiają do repo)

| Nazwa sekretu | Przeznaczenie (bez wartości) |
|---|---|
| `APP_REPOSITORY` | pełna nazwa prywatnego repo aplikacji (`owner/nazwa`) |
| `APP_DEPLOY_KEY` | klucz prywatny deploy key (read-only) repo aplikacji |
| `DATABASE_URL` | produkcyjna baza — endpoint pooler (runtime collectora) |
| `X_SUPER_CLIENT` | wartość nagłówka `X-Super-Client` community API |
| `X_SUPER_CONTACT` | wartość nagłówka `X-Super-Contact` community API |

Workflow mapuje `X_SUPER_*` na zmienne środowiskowe `GWM_SUPER_CLIENT`/`GWM_SUPER_CONTACT`
wymagane przez klienta community API.

## ⚠️ Harmonogram a 60 dni bez aktywności

GitHub **automatycznie wyłącza** scheduled workflows w publicznym repo po
**60 dniach braku aktywności w repo** (dokumentacja: „Disabling and enabling
a workflow"). Workflow przestaje się uruchamiać — to nie jest błąd polla.

**Procedura ponownego włączenia** (dowolna z opcji):

- GitHub UI: repo → zakładka **Actions** → workflow „collector" → **Enable workflow**;
- CLI: `gh workflow enable collector.yml` w tym repo;
- dowolny push do tego repo (resetuje 60-dniowe okno aktywności).

**Mitygacja:** comiesięczna kontrola w kalendarzu (sprawdź Actions; kliknij
„Enable workflow", jeśli wyłączone) albo dowolny push raz w miesiącu. Opcjonalny
automatyczny „keepalive" (drugi workflow robiący pusty commit raz w miesiącu)
wymaga dodania drugiego pliku workflow — decyzja właściciela, poza bieżącym
zakresem.

Uwaga: harmonogramy GitHub bywają **opóźniane** pod obciążeniem (kilka–kilkanaście
minut) — realne odstępy między pollami nie muszą wynosić dokładnie 5 minut.

## Bezpieczeństwo

- Triggery wyłącznie `schedule` + `workflow_dispatch` — sekrety nie są dostępne
  dla `pull_request` z forków; nikt z zewnątrz nie uruchomi workflow z sekretami.
- Deploy key: **read-only**, przypisany do **jednego** repo, bez powiązania z
  kontem właściciela.
- Rotacja deploy key: wygeneruj nową parę → dodaj publiczną połowę do repo
  aplikacji (Settings → Deploy keys) → podmień sekret `APP_DEPLOY_KEY` → usuń
  starą połowę z repo aplikacji.
- W tym repo nie ma kodu aplikacji, plików `.env`, ścieżek lokalnych ani danych
  osobowych.
