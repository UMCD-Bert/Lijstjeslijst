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
8. **Elk `<input>`/`<select>`/`<textarea>` moet `font-size: 16px` of groter hebben.** Onder de
   16px zoomt mobiele Safari automatisch in zodra je het veld aantikt, en de pagina blijft daarna
   ingezoomd/verschuifbaar ("venster is breder dan scherm") tot de gebruiker handmatig weer
   uitzoomt — dit is puur CSS-gedreven en valt NIET op in de Claude Browser-testtool hier (Chromium-
   gebaseerd, repliceert dit Safari-specifieke gedrag niet), dus alleen een `grep -n "font-size: 0\."`
   op input/select/textarea-regels in `index.html` vangt het. (Aanleiding: meerdere bestaande
   velden — zoekbalk, sorteer-select, bewerk-/toevoeg-formulieren — stonden op 13–15px; gefixt in
   v1.4.1, 2026-09-13.)

## Datamodel (kern) — generiek velden-systeem
- `lijsten`: naam, omschrijving, omslag_url, volgorde, `type_label` (vrije tekst, getoond als pill),
  `extra_velden` (jsonb array van `{key,label}`, max 4 — tekstvelden per item), `vink_velden` (jsonb
  array van `{key,label}`, max 5 — checkboxvelden per item), `doorstrepen` (boolean, default true —
  bepaalt of het aanvinken van het EERSTE vinkje het item doorstreept; staat op `false` bij
  Boeken/Muziek/Strips/Bordspellen omdat een verzameling geen mancolijst is, op `true` bij
  Lijst/Aangepast), `auto_import` (`openlibrary`/`musicbrainz`/`discogs`/`bgg`/null — `bgg` nog niet
  gebouwd), `bgg_username` (alleen relevant zodra BGG-import gebouwd wordt), `discogs_username`
  (alleen relevant bij `auto_import: 'discogs'`), `genest` (boolean, default false — generieke
  aan/uit-schakelaar voor een echte 1:n-structuur reeks→albums, zie hieronder; Strips-, Boeken- én
  Muziek-preset zetten 'm standaard aan, geen UI-toggle voor andere lijsttypes gebouwd want niet
  gevraagd).
- `lijst_items`: lijst_id (FK, on delete cascade), titel, `extra` (jsonb, matcht keys uit
  `extra_velden`), `vinkjes` (jsonb, matcht keys uit `vink_velden`), `omslag_url` (foto per item,
  los van de omslagfoto van de lijst), `bron` + `extern_id` (herkomst bij auto-import, voorkomt
  dubbele import), `reeks_id` (FK → `reeksen`, on delete cascade, alleen gebruikt bij `genest`
  lijstjes), volgorde.
- `reeksen` (2026-09-13, voor `genest` lijstjes): eigen tabel i.p.v. een tekstveld — `lijst_id` (FK,
  on delete cascade), `naam`, `omslag_url` (nog niet gebruikt in UI), `volgorde`. Puur generiek
  concept: een reeks kan een auteur zijn, maar net zo goed een boekenserie (bv. reisgidsen) waar de
  auteur juist niet relevant is — de gebruiker kiest zelf de naam, er zit geen vast "type" achter.
  Een `genest` lijstje toont ÉÉN gedeelde tabel (kolomkoppen dus maar 1x, niet per reeks herhaald —
  eerdere aanpak met een aparte tabel per reeks werd hierop afgekeurd: "kolomnamen per auteur kost
  veel te veel ruimte"), met per reeks een kop-rij (in-/uitklap-driehoekje, rename, itemaantal, evt.
  Open Library-/MusicBrainz-zoekicoon, afhankelijk van `auto_import` — zie reeksImportSources in de
  code, verplaats/verwijder — verwijderen cascadeert naar de items erin) gevolgd
  door (als niet ingeklapt) de items van die reeks. De eerdere "aparte tabel per reeks"-opzet loste
  toen wel een ander probleem op (mini-formulieren per reeks die op iPhone een te smalle
  horizontaal-scrollbare strook gaven) — dat probleem keert niet terug omdat toevoegen inmiddels via
  de ÉÉN gedeelde "+ Toevoegen"-knop gaat (zie verderop), niet meer via per-reeks formulieren in de
  tabel. In-/uitklappen is puur UI-state (`state.collapsedReeks`, niet in de DB) en wordt genegeerd
  zodra er een actieve zoekopdracht of filter is (anders zou een ingeklapte reeks zoekresultaten
  verbergen). Toevoegen gaat via ÉÉN
  gedeelde "+ Toevoegen"-knop onderaan de hele lijst (niet per reeks) — opent een klein formulier
  met titel + een reeks-`<select>` (bestaande reeksen + een "+ Nieuwe reeks…"-optie die een naamveld
  toont); dit verving eerdere losse mini-formulieren per reeks plus een apart "+ Nieuwe reeks"-vak,
  wat samen te veel altijd-zichtbare UI was voor een handeling die zelden gebeurt. Zoeken matcht ook
  op reeksnaam. Items zonder `reeks_id` (zou niet via de UI moeten ontstaan) worden alsnog getoond
  onder een niet-verwijderbare "Zonder reeks"-kop, als vangnet. **Fix (2026-09-13): reeksen zonder
  match onder een actieve zoekopdracht/filter worden nu volledig verborgen** i.p.v. getoond met een
  "Geen items met dit filter"-placeholder — bij bv. zoeken op "queen" bleven voorheen ALLE reeksen
  (ook artiesten zonder enige treffer) als lege kop-rij zichtbaar, wat het geen bruikbaar filter
  maakte. Alleen als een reeks zelf 0 treffers heeft ÉN er geen actieve query is (het "leeg"-label
  voor lege reeksen tijdens gewoon beheer) blijft-ie zichtbaar.
- Detailweergave-header (2026-09-13 herzien voor schermeconomie): terug-pijl, titel, "Velden
  bewerken" (tandwiel-icoon) en omslagfoto-toevoegen (camera-icoon, alleen zonder cover) staan alle
  vier op ÉÉN compacte regel i.p.v. losse rijen erboven/eronder. Omschrijving is verplaatst ván de
  hoofdweergave náár binnen het "Velden bewerken"-paneel (bleek voor deze gebruiker geen
  toegevoegde waarde te hebben als altijd-zichtbaar element). Zoeken, sorteren én de "Filters"-knop
  (zie hieronder) staan samen op één regel i.p.v. gestapeld. Reden: bij meerdere vinkjes/velden
  (Boeken had bv. auteur-tekstveld + 4 vinkjes) stond er een lange muur van chrome vóórdat de eerste
  daadwerkelijke inhoud zichtbaar werd — deze herziening scheelt volgens de gebruiker "50% of meer
  van de schermeconomie".
- Filters (per-vinkje Alles/Wel/Niet) staan net als "Velden bewerken" standaard ingeklapt achter een
  "Filters"-knop (2026-09-13) — klapt vanzelf open als er al een actief filter staat.
- Snelkeuzes (Lijst/Boeken/Muziek/Strips/Bordspellen/Aangepast) vullen bij aanmaken alleen de
  velden hierboven vooraf in — daarna is alles per lijstje los aan te passen via "Velden bewerken".
  Nieuwe types toevoegen is meestal een kleine JS-wijziging (preset), geen migratie.
- Boeken-sjabloon (2026-09-13 herzien): `genest: true`, geen los "Auteur"-tekstveld meer — groeperen
  gebeurt via `reeksen` (zie hierboven). Blijft: 4 vinkjes (E-book, Fysiek boek, Gelezen door Ellen,
  Gelezen door Bert) — geen generieke "In bezit" meer, want e-book/fysiek dekken bezit al specifieker.
  Aanleiding voor de herziening: de Open Library-auteur-import (zie hieronder) was op zichzelf
  onbruikbaar — één auteur levert vaak meerdere losse OL-auteursrecords op (verplicht steeds opnieuw
  kiezen + naam retypen bij een misser, geen weg terug naar de kandidatenlijst), resultaten stonden
  standaard allemaal aangevinkt (i.p.v. andersom), en de NL-dekking is te onvolledig om als enige
  invoerpad te dienen (bv. maar 1 van de 5 in bezit zijnde James Norbury-boeken vindbaar). Open
  Library-zoeken bestaat nu als *secundair* hulpmiddel binnen een al aangemaakte reeks (zoekicoon in
  de reeks-kop) i.p.v. als enige invoerpad: opent direct met de reeksnaam als zoekterm (geen
  hertypen), toont een "← Andere kandidaat proberen"-link (geen volledige reset meer bij een
  verkeerde auteurstreffer), en toont resultaten standaard NIET aangevinkt (aanvinken = toevoegen,
  i.p.v. moeten uitvinken uit tientallen ongewenste titels). Discogs/MusicBrainz-imports (Muziek)
  blijven ongewijzigd all-checked, want die importeren een hele bestaande collectie i.p.v. een
  "blader door het hele oeuvre"-lijst.
- Boeken-import (Open Library) filtert op Nederlandstalige edities via
  `search.json?q=author_key:{id} AND language:dut&editions.language=dut&fields=key,title,cover_i,
  editions,editions.title,editions.cover_i` (i.p.v. de taal-agnostische `/authors/{id}/works.json`,
  die geen taalveld ÉN geen cover-ID teruggeeft). **Val opgelost (2026-09-12): het `language=dut`-
  filter als los queryparameter (i.p.v. in `q=`) filtert alleen mee welke WERKEN een Nederlandse
  editie hébben — de teruggegeven `title`/`cover_i` blijven die van de standaard-/eerste editie
  (meestal Engels, bv. "It" i.p.v. "Het").** De juiste vorm combineert een Solr-filter in `q=`
  (`author_key:X AND language:dut`) mét de `editions`-subexpansie (`editions.language=dut` +
  `editions.title`/`editions.cover_i` in `fields=`) om de daadwerkelijke Nederlandse editie (titel
  én cover) uit `docs[].editions.docs[0]` te lezen, met de werk-titel/cover als fallback als er
  onverhoopt geen editions-match is. `omslag_url` wordt bij import direct meegezet op het item.
  **Tweede val opgelost (2026-09-13):** sommige werken hebben in Open Library helemaal geen
  taalmetadata op hun editie, zelfs als de titel zelf al Nederlands is (bv. "Grote Panda & Kleine
  Draak" van James Norbury) — de strikte `language:dut`-query levert dan 0 resultaten terwijl het
  boek wel bestaat. Fallback: bij 0 resultaten alsnog een ongefilterde `author_key`-query tonen
  (alle werken van de auteur, ongeacht taal) met een duidelijke melding dat de titels mogelijk niet
  Nederlands zijn — beter een te ruime lijst waaruit de gebruiker zelf kiest dan een dichtgetimmerd
  "niets gevonden".
- Muziek-sjabloon (2026-09-13, "artiest = reeks = auteur"): zelfde `genest`-aanpak als Boeken —
  `genest: true`, geen los "Artiest"-tekstveld meer, groeperen gebeurt via `reeksen` (elke reeks =
  één artiest). Vinkjes: In bezit, plus LP en CD naast elkaar (los van elkaar aan te vinken, want
  eenzelfde album kan in beide formaten in bezit zijn) — toegevoegd op uitdrukkelijk verzoek, want
  een collectie bevat beide fysieke vormen en dat onderscheid moet filterbaar blijven. Discogs'
  collection-endpoint geeft het/de formaat(en) van een release al mee in dezelfde call als de rest
  (`basic_information.formats[].name`, bv. "Vinyl"/"CD") — geen aparte aanroep per release nodig.
  Bestaande 162 items gemigreerd: 88 reeksen (één per unieke artiestwaarde) aangemaakt, LP/CD per
  item teruggehaald uit de live Discogs-collectie en ingevuld. De Discogs-bulkimport (importeert in
  één keer de hele collectie, geen per-reeks handeling) matcht of maakt voortaan zelf een reeks per
  artiestnaam en zet LP/CD automatisch op basis van het Discogs-formaat — geen handwerk bij een
  nieuwe import. De MusicBrainz-zoekicoon in de reeks-kop (voor wishlist-items die je nog niet in
  bezit hebt) werkt nu net als bij Boeken/Strips: zoekt direct op de artiestnaam van die reeks.
- Items worden getoond als een tabel (`.items-table` in `.items-scroll`, `overflow-x:auto`):
  titelkolom sticky links, vinkjes-koppen (E-book/Auteur/etc.) ÉÉN keer bovenaan i.p.v. per rij
  herhaald, actieskolom (foto/bewerk/verwijder) sticky rechts. Tabelbreedte wordt expliciet in JS
  gezet (`180 + aantal_vinkjes * 54 + 90` px) i.p.v. CSS `min-width:max-content` — die laatste
  dwong ALLE tekst in de tabel tot één regel (titelkolom liep op tot 9000px), een echte val bij
  scrollbare tabellen: gebruik een berekende pixelbreedte, niet `max-content`, als je select
  kolommen vast moet houden terwijl andere mogen wrappen.
- App-header (logo/tagline/"Nieuw lijstje") verdwijnt in de lijst-detailweergave (alleen "← Alle
  lijstjes" + lijstnaam) — was op iPhone te veel verloren ruimte vóór je daadwerkelijk items ziet.
  Lege omslagfoto-placeholder in detailweergave vervangen door een smal "+ Omslagfoto toevoegen"-
  linkje i.p.v. een grote lege blok van 140px+.
- Zoeken (op titel + alle tekstvelden) en sorteren (Handmatig/Titel/per tekstveld) per lijstje,
  boven de items. Bij een actieve sortering verdwijnen de handmatige verplaats-pijltjes.
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
  automatisch invullen: dat zit achter BGG's `thing`-endpoint. **Update (2026-09-12): BGG vereist
  sinds kort verplichte app-registratie + Bearer-token voor de HELE XML-API (v1, v2, GraphQL)** —
  geverifieerd via directe test (401 + `WWW-Authenticate: Bearer`), bevestigd door BGG's eigen
  `/using_the_xml_api`-pagina. Applicatie is aangevraagd (non-commercial, app-URL = live Lijster-
  site) — goedkeuring kan volgens BGG een week of langer duren. Zodra het token er is: token in de
  client-JS zetten (onvermijdelijk zichtbaar in broncode bij een statische app zonder backend — BGG
  noemt dat zelf een aanvaard risico, vergelijkbaar met de storage-orphan-trade-off hierboven) en
  pas dan de import bouwen. Async-gedrag (BGG kan 202 teruggeven terwijl de export wordt
  voorbereid) vraagt om een retry-met-backoff bij het ophalen.
- MusicBrainz-auto-import (Muziek) kon vanuit deze dev-omgeving niet betrouwbaar getest worden
  ("server is busy"-responses, waarschijnlijk rate-limiting op het dev-IP) — nog niet bevestigd dat
  dit vanaf een telefoon/thuisnetwerk wél werkt.
- ~~Cover-afbeelding automatisch ophalen bij Boeken-import~~ — gebouwd (v1.2.1): `search.json` geeft
  `cover_i` gewoon mee in dezelfde aanroep als de taalfilter, geen aparte aanroep per boek nodig.
- Muziek: check of de labels (filter "Niet", kaart-samenvatting) voor Muziek specifiek "Wishlist"
  moeten zeggen i.p.v. het generieke "Niet" — het bestaande "In bezit"-vinkje dekt het wishlist/
  in-bezit-gedrag zelf al (aanvinken bij aankoop = van wishlist naar in bezit), dit is puur een
  tekst/UI-vraag.
- ~~Muziek-collectie inlezen vanaf Discogs~~ — wordt gebouwd (2026-09-12). Verificatie vooraf: geen
  auth nodig voor een publieke collectie, CORS staat open (`Access-Control-Allow-Origin: *`, alleen
  zichtbaar als de request een `Origin`-header meestuurt — vanuit een browser dus altijd, curl
  zonder `-H Origin` liet dit ten onrechte lijken op een blokkade), geen custom User-Agent vereist
  (in tegenstelling tot wat eerder aangenomen werd), rate limit 25 req/min onbevestigd — ruimschoots
  genoeg. `cover_image` zit al in de collection-respons, geen aparte aanroep nodig. Gebruikersnaam:
  "bertellen" — collectie stond eerst op privé (401), inmiddels op publiek gezet. Nieuwe
  DB-kolom `lijsten.discogs_username` toegevoegd. `auto_import: 'discogs'` is een los te kiezen
  Muziekbron naast `musicbrainz` (schakelaar in "Velden bewerken"), niet de nieuwe preset-default —
  MusicBrainz-zoeken blijft nuttig voor wishlist-items die nog niet in bezit zijn.
- Barcode/ISBN-scan via de camera (2026-09-12, wens voor later — geen GO): het handmatige
  ISBN-invoerpad voor Boeken is inmiddels gebouwd (zie hieronder, 2026-09-13) — dit restpunt gaat
  nu alleen nog over het automatisch ÍNLEZEN van die code via de camera (en, voor platen, barcode
  i.p.v. ISBN — Discogs heeft ook een barcode-zoekfunctie op releases). Nog te ontwerpen: welke
  barcode-scanbibliotheek (bv. browser-native `BarcodeDetector` API vs. een JS-library als ZXing,
  i.v.m. browserondersteuning op iPhone Safari).
- ~~Boeken: toevoegen op ISBN~~ — gebouwd (2026-09-13). Los "+ Boek via ISBN"-knopje naast
  "+ Toevoegen", alleen bij `auto_import: 'openlibrary'` (dus vooralsnog specifiek Boeken). Gebruikt
  Open Library's `api/books?bibkeys=ISBN:...&jscmd=data` (één call geeft titel, auteur ÉN cover
  direct terug, i.p.v. losse author-lookup zoals bij de reeks-import). Toont een preview (cover,
  titel, auteur) met een voorgestelde reeks — matcht automatisch op een bestaande reeks als de
  gevonden auteursnaam overeenkomt, anders "+ Nieuwe reeks…" voorgevuld met die auteursnaam, allebei
  door de gebruiker nog aan te passen vóór bevestigen. Dedupe op `extern_id` (het ISBN): een al
  toegevoegd ISBN geeft direct een duidelijke melding i.p.v. een dubbel item.
- ~~Strips: geneste reeks→albums-structuur~~ — gebouwd (2026-09-13). Was eerst een plat
  `reeks`-tekstveld per item (elk album herhaalde de reeksnaam apart); nu een echte 1:n-relatie:
  nieuwe tabel `reeksen` + `lijst_items.reeks_id`, generieke `lijsten.genest`-schakelaar (zie
  Datamodel hierboven). Enige bestaande Strips-lijst ("Strips Bert", 1 item/reeks "Suske en
  Wiske") gemigreerd.
  ~~Vervolg (2026-09-13): iPhone-scrollbug + verwarrende "nieuw album"-knop~~ — ook opgelost, bij
  dezelfde herbouw als de Boeken-herstructurering hieronder (elke reeks kreeg een eigen tabel +
  los add-formulier i.p.v. alles samengeperst in één gedeelde tabel). Muziek-label-vraag hieronder
  blijft nog los staan.
- ~~Boeken: herstructurering rond auteur/reeks + Open Library-import~~ — gebouwd (2026-09-13, zie
  Datamodel/Boeken-sjabloon hierboven voor het volledige waarom). Kern: `genest: true`, auteur (of
  serienaam, bv. reisgidsen) is nu een reeks i.p.v. een tekstveld; Open Library-zoeken is secundair
  hulpmiddel per reeks geworden i.p.v. het enige, onbetrouwbare invoerpad. Bestaande data
  gemigreerd: reeksen "James Norbury" en "Charlie Mackesy" aangemaakt voor de 3 bestaande items.
  ~~Vervolg (2026-09-13): schermeconomie~~ — n.a.v. concrete layout-feedback op deze herstructurering
  (screenshot + puntsgewijze wensen) ook: header-regel gecombineerd (zie Datamodel hierboven),
  omschrijving verplaatst naar "Velden bewerken", zoeken/sorteren/filters op één regel, per-reeks
  mini-formulieren + los "Nieuwe reeks"-vak vervangen door één "+ Toevoegen"-knop met reeks-kiezer,
  en reeksen zijn nu in-/uitklapbaar (met itemaantal in de kop) — geldt ook voor Strips, zelfde
  gedeelde code.
- ~~Muziek: reeks/auteur-structuur doorgevoerd~~ — gebouwd (2026-09-13, zie Datamodel/
  Muziek-sjabloon hierboven). Zelfde `genest`-aanpak als Boeken/Strips, artiest = reeks. Op
  verzoek ook meteen het LP/CD-onderscheid toegevoegd (twee losse vinkjes naast "In bezit"),
  automatisch gevuld uit de Discogs-collectie bij de migratie en voortaan ook automatisch gezet
  bij nieuwe Discogs-imports. Discogs-bulkimport groepeert nieuwe albums voortaan automatisch in
  de juiste (of een nieuwe) reeks per artiest.
