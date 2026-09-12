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

## Datamodel (kern)
- `lijsten`: naam, omschrijving, sjabloon (`simpel`/`boeken`/`muziek`), omslag_url, volgorde
- `lijst_items`: lijst_id (FK, on delete cascade), titel, subtitel (auteur/artiest, ongebruikt bij
  `simpel`), afgevinkt (alleen betekenisvol bij `simpel`), volgorde
- Generieke `titel`/`subtitel`-kolommen i.p.v. aparte auteur/artiest-kolommen per sjabloon — het
  sjabloon op de lijst bepaalt hoe de UI die twee kolommen labelt en rendert.
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
