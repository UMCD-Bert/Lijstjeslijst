# Lijster — projectcontext voor Claude Code

## Wat dit is
Gedeelde lijstjes-app voor de eigenaar en zijn partner (Ellen): boeken, muziek, en losse lijstjes
met een aanpasbaar veldenschema per lijstje. In de app en op het beginscherm heet hij **"Lijster"**
(gestileerd vogeltje als icoon) — de repo/URL heten nog "Lijstjeslijst" (bewust niet hernoemd, dat
zou het al geïnstalleerde beginscherm-icoon van de eigenaar/Ellen breken). Zelfde bouwstijl als de
andere apps: één `index.html` (vanilla JS) + Supabase-backend, gehost op GitHub Pages, gedeelde
toegang (geen aparte accounts per persoon).

- GitHub: repo `Lijstjeslijst` onder account `UMCD-Bert` → live op https://umcd-bert.github.io/Lijstjeslijst/
- Supabase: project **"Verzamelingen"** (`dckojtvyxqsgvcfgufpf`, regio eu-west-2) — gekozen omdat
  boeken/platen-lijstjes qua aard dichter bij de collectie-apps (Pokémon-kaarten, tassen, items) staan
  dan bij de huishouddata in het "Huishouden"-project (Budget, Gezin). Eigen tabellen (`lijsten`,
  `lijst_items`), geen relatie met de bestaande verzameltabellen.
- Omslagfoto's (lijst- én itemniveau): bestaande publieke bucket `afbeeldingen` (prefix
  `lijstjeslijst/`) — geen nieuwe bucket aangemaakt. Er is geen anon-delete-policy op die bucket;
  een verwijderde/vervangen foto laat een ongebruikt bestand achter in storage (bewuste,
  lage-impact trade-off).
- GitHub Pages vereist de letterlijke bestandsnaam `index.html`

## Werkafspraken — ALTIJD aanhouden
1. Wacht op expliciete GO voordat je gaat coderen, ook bij kleine/directe instructies. De eigenaar
   somt wensen soms één voor één op zonder GO — dan alleen loggen in dit bestand, niet bouwen, tot
   hij expliciet "ga bouwen" o.i.d. zegt.
2. SQL-wijzigingen eerst als los, copy-pastebaar blok tonen en om goedkeuring vragen — na akkoord
   voert Claude ze zelf uit via de Supabase MCP-connector (niet de eigenaar in de SQL-editor).
3. Na elke niet-triviale JS-wijziging: minstens een korte handmatige smoke-test in de browser tegen
   de live Supabase-data (geen jsdom-suite hier); test-lijstjes/items die daarbij ontstaan direct
   weer opruimen (via de UI of `delete from lijsten/lijst_items where ...`).
4. Vaste kwaliteitscontrole vóór elke levering: versienummer verhoogd + expliciet genoemd
   (`APP_VERSION` in `index.html`, zichtbaar onderaan de pagina), service-worker `CACHE_VERSION`
   ook opgehoogd bij elke inhoudelijke wijziging, zelf-check tegen deze werkafspraken.
5. Kleine backlogpunten mogen automatisch mee in de eerstvolgende bouwronde.
6. Wees terughoudend met live tests tegen externe API's (Open Library/MusicBrainz/BGG/Discogs)
   vanuit deze dev-omgeving — herhaalde aanroepen triggerden al Cloudflare-blokkades/rate-limits.
   Eén gerichte verificatie is genoeg; niet blijven herhalen.
7. **Geen enkel interactief element mag alleen bereikbaar zijn via `:hover`/`:focus-within`** (bv.
   `opacity: 0` die pas bij hover naar `1` gaat) — op een touchscreen bestaat hover niet, dus zo'n
   knop is daar onzichtbaar én onbereikbaar. Dit soort bugs valt NIET op via de gebruikelijke
   browser-testflow hier: geautomatiseerde clicks op coördinaten werken ook als het element
   opacity:0 heeft, dus een geslaagde test in deze omgeving bewijst niets over bruikbaarheid op een
   telefoon. Vaste stap vóór levering van UI-wijzigingen: `grep -n "opacity: 0"` (en vergelijkbare
   hover-only-reveal patronen) door `index.html` en handmatig nalopen of elk zo'n element ook zonder
   hover/focus bereikbaar is. (Aanleiding: v1.1.0 verstopte de bewerk/foto/verwijder-knoppen per item
   en de verplaats/verwijder-knoppen per lijstje volledig achter `:hover`, onbruikbaar op iPhone.)

## Datamodel (kern) — generiek velden-systeem
- `lijsten`: naam, omschrijving, omslag_url, volgorde, `type_label` (vrije tekst, getoond als pill),
  `extra_velden` (jsonb array van `{key,label}`, max 4 — tekstvelden per item), `vink_velden` (jsonb
  array van `{key,label}`, max 5 — checkboxvelden per item), `doorstrepen` (boolean, default true —
  bepaalt of het aanvinken van het EERSTE vinkje het item doorstreept; staat op `false` bij
  Boeken/Muziek/Strips/Bordspellen omdat een verzameling geen mancolijst is, op `true` bij
  Lijst/Aangepast), `auto_import` (`openlibrary`/`musicbrainz`/`bgg`/null — `bgg` nog niet gebouwd),
  `bgg_username` (alleen relevant zodra BGG-import gebouwd wordt).
- `lijst_items`: lijst_id (FK, on delete cascade), titel, `extra` (jsonb, matcht keys uit
  `extra_velden`), `vinkjes` (jsonb, matcht keys uit `vink_velden`), `omslag_url` (foto per item,
  los van de omslagfoto van de lijst), `bron` + `extern_id` (herkomst bij auto-import, voorkomt
  dubbele import), volgorde.
- Snelkeuzes (Lijst/Boeken/Muziek/Strips/Bordspellen/Aangepast) vullen bij aanmaken alleen de
  velden hierboven vooraf in — daarna is alles per lijstje los aan te passen via "Velden bewerken".
  Nieuwe types toevoegen is meestal een kleine JS-wijziging (preset), geen migratie.
- Boeken-sjabloon: 1 tekstveld (Auteur) + 4 vinkjes (E-book, Fysiek boek, Gelezen door Ellen,
  Gelezen door Bert) — geen generieke "In bezit" meer, want e-book/fysiek dekken bezit al specifieker.
- Boeken-import (Open Library) filtert op Nederlandstalige edities via
  `search.json?author_key=...&language=dut` (i.p.v. de taal-agnostische `/authors/{id}/works.json`,
  die sowieso geen taalveld teruggeeft) — geeft ook meteen een schonere, kleinere titellijst.
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
- Losstaand van dit project: de bestaande `Verzamelingen`-tabellen hadden RLS uitgeschakeld — dit is
  inmiddels gefixt (RLS aan + permissieve policies, zelfde toegangsmodel als voorheen).
- BGG-collectie-import voor Bordspellen (auto_import: 'bgg'): username "bertuf", inclusief
  uitbreidingen. Ontwerp uitgewerkt maar nog niet gebouwd: BGG's `collection`-endpoint (met
  `stats=1`) geeft titel + BGG-id + rank in één call — genoeg voor import én een latere "Ververs
  BGG-ranks"-knop (herhaalt dezelfde call, matcht op `extern_id`). Ontwerper/uitgever bewust NIET
  automatisch invullen: dat zit achter BGG's `thing`-endpoint, die tijdens testen herhaaldelijk door
  Cloudflare werd geblokkeerd vanuit deze dev-omgeving — pas automatiseren als blijkt dat het wél
  werkt vanaf een telefoon/thuisnetwerk. Async-gedrag (BGG kan 202 teruggeven terwijl de export
  wordt voorbereid) vraagt om een retry-met-backoff bij het ophalen.
- MusicBrainz-auto-import (Muziek) kon vanuit deze dev-omgeving niet betrouwbaar getest worden
  ("server is busy"-responses, waarschijnlijk rate-limiting op het dev-IP) — nog niet bevestigd dat
  dit vanaf een telefoon/thuisnetwerk wél werkt.
- Cover-afbeelding automatisch ophalen bij Boeken-import (Open Library): `works.json`/`search.json`
  geven geen cover-ID mee, dus dit vergt een aparte aanroep per boek (editions-lookup) — risico op
  dezelfde soort blokkades als bij BGG/MusicBrainz. Nog niet gebouwd; de handmatige foto-upload per
  item (camera-icoontje per regel) is er al, dat dekt de behoefte voorlopig.
- Muziek: check of de labels (filter "Niet", kaart-samenvatting) voor Muziek specifiek "Wishlist"
  moeten zeggen i.p.v. het generieke "Niet" — het bestaande "In bezit"-vinkje dekt het wishlist/
  in-bezit-gedrag zelf al (aanvinken bij aankoop = van wishlist naar in bezit), dit is puur een
  tekst/UI-vraag.
- Muziek-collectie inlezen vanaf **Discogs** (ze hebben een account) — zelfde patroon als de
  BGG-import hierboven: collectie-endpoint (`api.discogs.com/users/{username}/collection/folders/0/
  releases`) kan zonder auth voor een publieke collectie, vereist een beschrijvende User-Agent (kan
  niet vanuit browser-fetch, zelfde beperking als MusicBrainz) en heeft rate limits. Nog niet
  getest op CORS/bereikbaarheid vanuit deze omgeving — eerst verifiëren voor het gebouwd wordt.
  Discogs-gebruikersnaam nog op te vragen zodra dit wordt opgepakt.
