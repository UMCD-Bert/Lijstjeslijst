# Lijstjeslijst — projectcontext voor Claude Code

## Wat dit is
Gedeelde lijstjes-app voor de eigenaar en zijn partner (Ellen): boeken, muziek, en losse lijstjes
(boodschappen, taken, etc.) met een vast sjabloon per lijstje. Zelfde bouwstijl als de andere apps:
één `index.html` (vanilla JS) + Supabase-backend, gehost op GitHub Pages, gedeelde toegang
(geen aparte accounts per persoon).

- GitHub: repo `Lijstjeslijst` onder account `UMCD-Bert` → live op https://umcd-bert.github.io/Lijstjeslijst/
- Supabase: project **"Verzamelingen"** (`dckojtvyxqsgvcfgufpf`, regio eu-west-2) — gekozen omdat
  boeken/platen-lijstjes qua aard dichter bij de collectie-apps (Pokémon-kaarten, tassen, items) staan
  dan bij de huishouddata in het "Huishouden"-project (Budget, Gezin). Eigen tabellen (`lijsten`,
  `lijst_items`), geen relatie met de bestaande verzameltabellen.
- Omslagfoto's: bestaande publieke bucket `afbeeldingen` (prefix `lijstjeslijst/`) — geen nieuwe
  bucket aangemaakt. Er is geen anon-delete-policy op die bucket; een verwijderde/vervangen foto
  laat een ongebruikt bestand achter in storage (bewuste, lage-impact trade-off).
- GitHub Pages vereist de letterlijke bestandsnaam `index.html`

## Werkafspraken — ALTIJD aanhouden
1. Wacht op expliciete GO voordat je gaat coderen, ook bij kleine/directe instructies.
2. SQL-wijzigingen eerst als los, copy-pastebaar blok tonen en om goedkeuring vragen — na akkoord
   voert Claude ze zelf uit via de Supabase MCP-connector (niet de eigenaar in de SQL-editor).
3. Na elke niet-triviale JS-wijziging: verifieer met een jsdom-simulatie (Supabase gemockt) vóór levering.
4. Vaste kwaliteitscontrole vóór elke levering: versienummer verhoogd + expliciet genoemd, gediffed
   tegen vorige versie, testresultaten herhaald bij levering, zelf-check tegen deze werkafspraken.
5. Kleine backlogpunten mogen automatisch mee in de eerstvolgende bouwronde.

## Datamodel (kern) — generiek velden-systeem (sinds v1.1, zie git-historie voor de oudere sjabloon-vaste versie)
- `lijsten`: naam, omschrijving, omslag_url, volgorde, `type_label` (vrije tekst, getoond als pill),
  `extra_velden` (jsonb array van `{key,label}`, max 4 — tekstvelden per item), `vink_velden` (jsonb
  array van `{key,label}`, max 3 — checkboxvelden per item), `auto_import` (`openlibrary`/
  `musicbrainz`/`bgg`/null), `bgg_username` (alleen gezet na een BGG-collectie-import).
- `lijst_items`: lijst_id (FK, on delete cascade), titel, `extra` (jsonb, matcht keys uit
  `extra_velden`), `vinkjes` (jsonb, matcht keys uit `vink_velden`), `bron` + `extern_id` (herkomst
  bij auto-import, voorkomt dubbele import), volgorde.
- Snelkeuzes (Lijst/Boeken/Muziek/Strips/Bordspellen/Aangepast) vullen bij aanmaken alleen de
  velden hierboven vooraf in — daarna is alles per lijstje los aan te passen via "Velden bewerken".
  Nieuwe types toevoegen is meestal een kleine JS-wijziging (preset), geen migratie.
- Volgorde wordt bijgehouden als timestamp (nieuw item/lijst = `Date.now()`); verplaatsen wisselt de
  `volgorde`-waarde van twee buren om (last-writer-wins, geen transacties nodig op deze schaal).

## Sync & offline
- Live sync via Supabase Realtime (`postgres_changes` op beide tabellen, wildcard event) — bij elke
  wijziging wordt de hele dataset opnieuw opgehaald (`refetchAll`), geen incrementele diff-logica.
  Werkt prima op deze schaal (huishoudelijke lijstjes, geen duizenden rijen).
- Service worker (`service-worker.js`) cachet alleen de app-shell (HTML/manifest/icons/supabase-js),
  read-only fallback bij geen verbinding — geen queue/sync-logica voor schrijfacties zoals bij Gezin.
  Schrijven zonder verbinding faalt gewoon met een alert; dat is bewust simpel gehouden.

## Platform
PWA via `manifest.json` + `apple-touch-icon.png` + service worker — "Zet op beginscherm" op iPhone
geeft een fullscreen appicoon zonder Safari-balk. Geen Claude-login nodig, geen native app.

## Bekende openstaande punten (geen GO — pas oppakken na expliciete instructie)
- Cover-foto's kunnen niet verwijderd worden uit Supabase Storage (geen anon-delete-policy op
  `afbeeldingen`) — alleen de databaseverwijzing wordt gewist.
- Losstaand van dit project: de bestaande `Verzamelingen`-tabellen (tcg_set_kaarten, lorcana_*,
  mtg_kaarten, sealed_producten, tassen(+fotos), items(+fotos), prijs_update_log) hebben RLS
  uitgeschakeld — volledig open voor iedereen met de anon-key. Niet aangeraakt vanuit dit project;
  los oppakken als de eigenaar dat wil.
- BGG-collectie-import voor Bordspellen (auto_import: 'bgg'): username "bertuf", inclusief
  uitbreidingen. Ontwerp al uitgewerkt maar nog niet gebouwd: BGG's `collection`-endpoint (met
  `stats=1`) geeft titel + BGG-id + rank in één call — genoeg voor import én een latere "Ververs
  BGG-ranks"-knop (herhaalt dezelfde call, matcht op `extern_id`). Ontwerper/uitgever bewust NIET
  automatisch invullen: dat zit achter BGG's `thing`-endpoint, die tijdens testen herhaaldelijk door
  Cloudflare werd geblokkeerd vanuit deze dev-omgeving (mogelijk bot-detectie op het IP, onduidelijk
  of dit ook vanaf een gewoon netwerk gebeurt) — pas automatiseren als blijkt dat het wél werkt vanaf
  een telefoon/thuisnetwerk. Async-gedrag (BGG kan 202 teruggeven terwijl de export wordt voorbereid)
  vraagt om een retry-met-backoff bij het ophalen.
- MusicBrainz-auto-import (Muziek) kon vanuit deze dev-omgeving niet betrouwbaar getest worden
  ("server is busy"-responses, waarschijnlijk rate-limiting op het dev-IP) — nog niet bevestigd dat
  dit vanaf een telefoon/thuisnetwerk wél werkt.
- Wensen van de eigenaar, nog te bouwen:
  - Boeken ophalen (Open Library-import): filteren op Nederlandstalige titels/edities, i.p.v. alle
    taalvarianten door elkaar tonen.
  - Geen doorstreep-styling bij het aanvinken van een "in bezit"-achtig vinkje (Boeken/Muziek/Strips/
    Bordspellen) — dat hoort bij een mancolijst/to-do, niet bij een verzameling. Doorstrepen mag wel
    blijven bij het "simpel"-sjabloon (`gedaan`). Vermoedelijk: alleen doorstrepen als de lijst maar
    één vinkje heeft én dat semantisch een to-do is — nader te bepalen hoe dit onderscheid gemaakt
    wordt (aparte vlag per vink-veld, of per sjabloon).
  - Boeken-sjabloon uitbreiden: formaat (e-book / fysiek boek — nog te bepalen of dit twee losse
    vinkjes wordt, of één keuzeveld; een boek kan in beide vormen aanwezig zijn) + twee losse
    "gelezen door"-vinkjes (Ellen, Bert) i.p.v. één generieke "gelezen". Samen met "In bezit" zijn dat
    3 vinkjes — past nog binnen de cap van 3, maar laat geen ruimte meer over voor extra vinkjes op
    dit sjabloon.
  - Cover-afbeelding per BOEK (niet per lijstje zoals nu): voorstel is albei — automatisch ophalen
    via Open Library's covers-API (`covers.openlibrary.org/b/id/{cover_id}-*.jpg`, cover_id zit al in
    de works.json-response van de auto-import) voor geïmporteerde boeken, met handmatige foto-upload
    (zelfde patroon als de bestaande lijst-omslagfoto) als terugvaloptie voor boeken die niet via
    Open Library zijn toegevoegd. Vereist opslag van een aparte afbeelding per item i.p.v. per lijst
    — grotere wijziging dan de andere punten hier.
