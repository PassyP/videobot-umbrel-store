# Videobot op Umbrel (community app store)

De app komt op de Umbrel **alleen via GitHub**: er is geen SSH, `scp` of `umbreld client`.
Installeren en updaten gaat volledig via de Umbrel-UI.

## Opbouw

```
deploy/umbrel/
  umbrel-app-store.yml          store-id "videobot"
  videobot-app/
    umbrel-app.yml              manifest (poort 8742, deterministicPassword, backupIgnore)
    docker-compose.yml          app_proxy (declaratie) + service `app` (image op digest gepind; `OWNER` en
                                digest zijn plaatsaanduidingen tot de eerste release)
    data/{db,media/...,tools,backups}/.gitkeep   vooraf aangemaakt, eigendom van uid 1000
```

Een community store is een GitHub-repository met `umbrel-app-store.yml` in de **root**. De inhoud
van `deploy/umbrel/` hoort dus in de root van een aparte store-repo. Het release-workflow
(`.github/workflows/release.yml`) doet dat zelf:

1. bouwt het multi-arch image (amd64 + arm64) en zet het op `ghcr.io/<eigenaar in kleine letters>/videobot`
   (de workflow zet de naam om naar kleine letters; Docker accepteert geen hoofdletters);
2. pint de digest in `docker-compose.yml` (de plaatsaanduiding `OWNER` wordt ingevuld), verhoogt de versie in
   `umbrel-app.yml` en vult `OWNER/REPO` in de links in;
3. opent een pull request in de app-repo (met ook `uv.lock` bij de wekelijkse yt-dlp-build);
4. job `store-sync`: kopieert `umbrel-app-store.yml` en `videobot-app/` (dezelfde gepinde bestanden, dezelfde
   digest) naar de root van de store-repo en pusht naar `main`.

**Eenmalige instelling voor stap 4** (app-repo, Settings, Secrets and variables, Actions):

| Wat | Waar | Waarde |
|---|---|---|
| `STORE_REPO` | Variables | `<gebruiker>/videobot-store` (de aparte store-repo, publiek, leeg aangemaakt, branch `main`) |
| `STORE_DEPLOY_KEY` | Secrets | de privé-sleutel van een ed25519-deploy key; de publieke sleutel staat in de store-repo onder Deploy keys met **Allow write access** |

Ontbreekt een van beide, dan slaat `store-sync` zichzelf over en blijft de workflow groen (handmatig
bijwerken: runbook 01, stap 8). Stap-voor-stap: `docs/runbooks/01-github-en-app-store.md`.

**Versienummer.** Een weekbuild begint bij de hoogste versietag in GHCR (nooit onder de versie in
`umbrel-app.yml`) en telt daar één patchnummer bij op, zodat een nog niet gemergde release-PR (bijv. 0.2.0 uit
een tag-build) niet door een lagere weekbuild wordt achterhaald; Umbrel biedt alleen een hogere versie aan.

**Release-PR en checks.** Een pull request die met de ingebouwde `GITHUB_TOKEN` wordt geopend, start geen andere
workflows, dus `ci.yml` draait er niet op. Zet daarom (optioneel) het secret `RELEASE_TOKEN` (een PAT of
app-token met schrijfrechten op contents en pull requests); de workflow gebruikt dat voor de PR en de checks
starten dan vanzelf. Zonder dat secret heeft het release-job `make check` al gedraaid vóór het bouwen; wil je
de CI-checks toch op de PR, sluit en heropen hem dan zelf.

**Icoon.** `umbrel-app.yml` verwijst met `icon:` naar `videobot-app/icon.svg` via een
`raw.githubusercontent.com`-URL. De release-stap vult `OWNER/REPO` in; de repository moet publiek zijn,
anders kan Umbrel het icoon niet ophalen.

**GHCR-pakket op Public.** Umbrel installeert via de UI en kan geen `docker login` doen. Zet na de eerste
release het pakket `videobot` op **Public** (GitHub, Packages, Package settings, Change visibility), anders
faalt de installatie met "unauthorized".

## Store toevoegen via de Umbrel-UI

1. Open de Umbrel in je browser en ga naar **App Store**.
2. Klik rechtsboven op **&#8943;** (drie puntjes) en kies **Community App Stores**.
3. Plak de GitHub-URL van de store-repo (bijv. `https://github.com/<jij>/<store-repo>`) en
   voeg toe.
4. Open de store "Videobot", kies **Videobot** en klik **Install**.
5. Open de app. De setupcode is het app-wachtwoord dat Umbrel bij de app toont
   (`deterministicPassword`). **Nog te verifiëren op het apparaat** of Umbrel dit als
   `defaultPassword` toont; zo niet, dan staat het in de app-instellingen.
6. Voer tijdens de setup je eigen bot-token in (gebruikersinvoer; het wordt nooit uit het
   Umbrel-seed afgeleid). De kill-switch staat bij de eerste start **aan**.

Updates: een nieuwe `version` in `umbrel-app.yml` verschijnt als update in de Umbrel-UI.
Rollback: publiceer de vorige digest met een hogere versie; dankzij `schema_compat` start
oudere code op een nieuwer schema.

## Back-ups en herstel

* De app maakt elk uur een geverifieerde SQLite-back-up in `data/backups` (48 uurlijks +
  14 dagelijks) en zet de secrets apart in `backups/secrets.json` (0600).
* Umbrel's eigen back-up neemt `backups/`, `media/processed/` (de wachtrij-bestanden, ca. 1,5 GB
  bij de defaults; retentie ruimt gepubliceerde bestanden na 7 dagen op), `thumbs/` en
  `library/` mee, maar niet `data/db`, `imports/` (geüploade exports: kunnen GB's zijn en zijn opnieuw te maken), `media/work` en `tools` (`backupIgnore`). Zonder
  `processed/` zou na een herstel elk wachtrij-item in Herstel belanden, want de bron-tweet kan
  inmiddels weg zijn.
* Is de database bij het opstarten leeg of weg, dan herstelt de app automatisch de nieuwste
  back-up. Daarbij gaat de kill-switch aan, worden due jobs `UNKNOWN`, en wordt een
  reconciliatiemelding klaargezet.
* Een herstel op een draaiende app kan niet: `docker exec ... vap restore --latest --force`
  laat de database ongemoeid, schrijft `data/db/RESTORE_REQUESTED` en meldt "Herstart de app in
  Umbrel om het herstel uit te voeren". De volgende start voert het herstel atomair uit
  (kopie naast de database, migratie + veiligheid daarop, dan pas verwisselen). De app neemt een
  slot (`data/db/vap.lock`): een tweede `vap run` weigert te starten.
* Na een herstel staat de kill-switch aan tot elke `UNKNOWN`-post is beoordeeld
  ("Alles: niet gepost" of per item); pas daarna kan hij uit.

## Discovery-API en de Umbrel-proxy

De discovery-engine op de Mac stuurt kandidaten naar `/api/v1/discovery` met een bearer-token (ADR 0009,
contract in `docs/discovery-api.md`). Zonder extra instelling zet Umbrel's `app_proxy` daar zijn eigen login voor.
De engine krijgt dan een HTML-loginpagina in plaats van VAP's JSON-antwoord. Daarom staat in
`videobot-app/docker-compose.yml`:

```yaml
app_proxy:
  environment:
    PROXY_AUTH_WHITELIST: "/api/*"
```

Het pad `/api/*` gaat dan langs Umbrel's login. VAP controleert het token zelf. Het dashboard blijft achter de
Umbrel-login en de VAP-login.

**Curl-check op het apparaat** (na elke release met deze wijziging). De poort staat in de app-info van Umbrel
(in deze repository `port: 8742` in `umbrel-app.yml`):

```sh
curl -i http://umbrel.local:<poort>/api/v1/discovery/config
```

Verwacht: `HTTP/1.1 401`, de header `WWW-Authenticate: Bearer realm="vap-discovery"` en een JSON-body
`{"error":{"code":"unauthorized",…}}`. Met een geldig token (`-H "Authorization: Bearer vapd_…"`) krijg je `200`
met de config. Zie je een HTML-loginpagina van Umbrel, dan werkt de whitelist niet.

**Fallback, alleen als de whitelist niet werkt**: zet onder `app_proxy` `PROXY_AUTH_ADD: "false"`. Umbrel's login
valt dan voor de hele app weg, ook voor het dashboard. Het dashboard wordt dan alleen beschermd door het eigen
VAP-wachtwoord (ADR 0007). Dat is een bewuste verzwakking van één laag. Het is een wijziging in `docker-compose.yml`
en loopt via dezelfde release-stap als de whitelist. Deze fallback is nog niet op een apparaat getest.

## Lokaal testen zonder Umbrel

`make umbrel-sim` bouwt het image en draait het onder dezelfde beperkingen als de compose
(uid 1000, `read_only`, tmpfs, `cap_drop ALL`, `no-new-privileges`). Zie `scripts/umbrel_sim.sh`.
