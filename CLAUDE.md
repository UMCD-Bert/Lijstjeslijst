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
6. Wees terughoudend met live tests tegen externe API's (Open Library/BGG/Discogs)
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
9. **Bij UI-wijzigingen rond de items-tabel: test ook op een smalle viewport (bv. 375px via
   `resize_window`/mobile-preset), niet alleen op standaardbreedte.** De tabel is met opzet breder
   dan een telefoonscherm kan zijn (zie hierboven, JS-berekende breedte) — een colSpan-rij (reeks-
   kop, bewerk-form, cover-/tracklist-zoekpaneel) erft die volle breedte, en kan zo op een iPhone
   gedeeltelijk buiten beeld vallen zonder dat dit op standaardbreedte zichtbaar is. Vaste stap:
   nieuwe/gewijzigde colSpan-inhoud ook smal testen, of geef 'm de `.wide-row-content`-klasse
   (sticky links + max-width op viewportbreedte) als 'ie flex-wrap-baar is. (Aanleiding: v1.17.1,
   gemeld door de gebruiker met een iPhone-screenshot — "Jaar van uitgave" en de kolomkop vielen af.)
10. **Twee sticky kolommen (bv. titel links + acties rechts) kunnen samen een smal scherm helemaal
    opvullen — dan zit alles ERTUSSEN (niet-sticky kolommen zoals vinkjes) permanent verstopt onder
    een van beide, hoe ver je ook scrolt.** Dit is een ANDER soort bug dan punt 9 hierboven (dat gaat
    over content die buiten beeld VALT/overflowt; dit gaat over content die op zijn scrollpositie
    exact onder een ondoorzichtige sticky kolom terechtkomt — optisch niet te onderscheiden van
    "leeg", dus ook op een correct geteste smalle viewport niet zichtbaar als bug tenzij je het
    checkt). Reken bij twee sticky kolommen expliciet uit of hun gecombineerde breedte de kleinste
    ondersteunde viewport nog ruimte laat, of meet het met `getBoundingClientRect()`/`clientWidth` in
    plaats van er alleen naar te kijken. (Aanleiding: v1.25.0, gevonden bij een grondige UX-review —
    sticky titelkolom + sticky actiekolom lieten op 375px nog maar 3px vrij, dus de vinkjeskolommen
    van Boeken/Muziek/Strips waren op een iPhone al sinds hun invoering volledig onbereikbaar zonder
    dat iemand dit als zodanig had gemeld of getest. Zelf inmiddels vervangen door een eenvoudigere
    aanpak, zie "Vervangen (2026-09-14, v1.26.0)" verderop bij de items-tabel.)
11. **Let op: sessies kunnen elkaars werk inhalen.** Meerdere Claude Code-sessies (of de eigenaar zelf
    rechtstreeks) kunnen op hetzelfde moment in deze repo werken. Als je bij het starten van een
    sessie een `APP_VERSION`/`CACHE_VERSION` in de code aantreft die hoger is dan het laatste dat in
    dit bestand gedocumenteerd staat, of code tegenkomt die inhoudelijk afwijkt van wat hier
    beschreven staat — dan heeft een andere sessie hier iets veranderd zónder dit bestand bij te
    werken (schendt zelf werkafspraak 4, maar gebeurt dus toch). Behandel de CODE dan als actuele
    waarheid, niet dit bestand: lees eerst wat er werkelijk staat (`git log`, `git show` op de
    onbekende commit(s)) vóórdat je verder bouwt, en werk dit bestand alsnog bij met wat je aantreft
    — anders raakt de documentatie steeds verder los van de werkelijke staat van de app. (Aanleiding:
    2026-09-14 — een andere sessie shipte v1.26.0 zonder CLAUDE.md aan te passen; deze sessie
    documenteerde tot dan toe nog de v1.25.0-aanpak die inmiddels alweer vervangen was.)

## Datamodel (kern) — generiek velden-systeem
- `lijsten`: naam, omschrijving, omslag_url, volgorde, `type_label` (vrije tekst, getoond als pill),
  `extra_velden` (jsonb array van `{key,label}`, max 4 — tekstvelden per item), `vink_velden` (jsonb
  array van `{key,label}`, max 5 — checkboxvelden per item), `doorstrepen` (boolean, default true —
  bepaalt of het aanvinken van het EERSTE vinkje het item doorstreept; staat op `false` bij
  Boeken/Muziek/Strips/Bordspellen omdat een verzameling geen mancolijst is, op `true` bij
  Lijst/Aangepast), `auto_import` (`openlibrary`/`discogs`/`bgg`/null — `bgg` nog niet gebouwd;
  `musicbrainz` bestond ook, maar is 2026-09-13 volledig verwijderd, zie Muziek-sjabloon hieronder),
  `bgg_username` (alleen relevant zodra BGG-import gebouwd wordt), `discogs_username`
  (alleen relevant bij `auto_import: 'discogs'`), `genest` (boolean, default false — generieke
  aan/uit-schakelaar voor een echte 1:n-structuur reeks→albums, zie hieronder; Strips-, Boeken- én
  Muziek-preset zetten 'm standaard aan, geen UI-toggle voor andere lijsttypes gebouwd want niet
  gevraagd).
- `lijst_items`: lijst_id (FK, on delete cascade), titel, `extra` (jsonb, matcht keys uit
  `extra_velden`), `vinkjes` (jsonb, matcht keys uit `vink_velden`), `omslag_url` (foto per item,
  los van de omslagfoto van de lijst), `bron` + `extern_id` (herkomst bij auto-import, voorkomt
  dubbele import), `reeks_id` (FK → `reeksen`, on delete cascade, alleen gebruikt bij `genest`
  lijstjes), `tracklist` (text, 2026-09-13, alleen bij Muziek) — bewust een los tekstveld i.p.v. een
  1:n-relatie zoals `reeksen`: het enige concrete doel was "op een track kunnen zoeken om te zien op
  welk album die staat", en de bestaande zoekfunctie (title + alle tekstvelden) doorzoekt dit veld al
  gratis mee zodra het meegenomen wordt in de haystack — een aparte `tracks`-tabel (met eigen
  CRUD/UI) zou dat doel niet beter dienen en is een wezenlijk andere relatievorm (kind van één item,
  niet een groepering van items zoals reeksen). Volgorde.
- `reeksen` (2026-09-13, voor `genest` lijstjes): eigen tabel i.p.v. een tekstveld — `lijst_id` (FK,
  on delete cascade), `naam`, `omslag_url` (nog niet gebruikt in UI), `volgorde`. Puur generiek
  concept: een reeks kan een auteur zijn, maar net zo goed een boekenserie (bv. reisgidsen) waar de
  auteur juist niet relevant is — de gebruiker kiest zelf de naam, er zit geen vast "type" achter.
  Een `genest` lijstje toont ÉÉN gedeelde tabel (kolomkoppen dus maar 1x, niet per reeks herhaald —
  eerdere aanpak met een aparte tabel per reeks werd hierop afgekeurd: "kolomnamen per auteur kost
  veel te veel ruimte"), met per reeks een kop-rij (in-/uitklap-driehoekje, rename, itemaantal, evt.
  evt. Open Library-zoekicoon (`secondarySearchSource(entry)`, sinds 2026-09-13 alleen nog Open
  Library — zie Muziek-sjabloon hieronder voor waarom MusicBrainz hier is verdwenen), verplaats/
  verwijder — verwijderen cascadeert naar de items erin) gevolgd
  door (als niet ingeklapt) de items van die reeks. De eerdere "aparte tabel per reeks"-opzet loste
  toen wel een ander probleem op (mini-formulieren per reeks die op iPhone een te smalle
  horizontaal-scrollbare strook gaven) — dat probleem keert niet terug omdat toevoegen inmiddels via
  de ÉÉN gedeelde "+ Toevoegen"-knop gaat (zie verderop), niet meer via per-reeks formulieren in de
  tabel. In-/uitklappen is puur UI-state (`state.collapsedReeks`, niet in de DB) en wordt genegeerd
  zodra er een actieve zoekopdracht of filter is (anders zou een ingeklapte reeks zoekresultaten
  verbergen). Toevoegen gaat via ÉÉN
  gedeelde "+ Toevoegen"-knop boven de itemlijst (niet per reeks) — opent een klein formulier
  met titel + een vrij tekstveld voor de reeks (auteur/artiest/serie), met bestaande reeksnamen als
  autocomplete-suggestie (`<datalist>`); dit verving eerdere losse mini-formulieren per reeks plus
  een apart "+ Nieuwe reeks"-vak, wat samen te veel altijd-zichtbare UI was voor een handeling die
  zelden gebeurt. **Herzien (2026-09-13, v1.16.0): geen expliciete keuze meer tussen "bestaande reeks
  kiezen" en "+ Nieuwe reeks…"** — de gebruiker hoeft niet meer bewust na te denken of een auteur/
  artiest al bestaat (expliciete aanleiding: bij "Artiest toevoegen" voor Dolly Parton moest eerst
  een aparte reeks worden aangemaakt vóór je één los album kon toevoegen, wat onintuïtief aanvoelde).
  De getypte naam matcht zelf, stil op de achtergrond, via `findOrCreateReeks()` (in
  `normalizeReeksNaam()`: lowercase, diacritics eruit, `&` ~ `en`, interpunctie genormaliseerd naar
  spaties) — "Suske en Wiske" en "Suske & Wiske" zijn zo dezelfde reeks, geen dubbele reeks per
  tikfoutje/spellingvariant. Dezelfde matcher wordt overal gebruikt waar een reeks/auteur/artiest
  automatisch gekoppeld wordt: de ISBN-import (auteursnaam) en de Discogs-bulkimport
  (artiestgroepering) — niet alleen de handmatige "+ Toevoegen"-form. Zoeken matcht ook op
  reeksnaam. Items zonder `reeks_id` (zou niet via de UI moeten
  ontstaan) worden alsnog getoond
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
  i.p.v. moeten uitvinken uit tientallen ongewenste titels). ~~De Discogs-bulkimport (Muziek) blijft
  ongewijzigd all-checked, want die importeert een hele bestaande collectie i.p.v. een
  "blader door het hele oeuvre"-lijst.~~ **Herzien (2026-09-14, v1.26.0):** ook de Discogs-bulkimport
  toont resultaten nu standaard NIET aangevinkt, met "Alles selecteren"/"Niets selecteren"-knoppen
  erbij (alleen zichtbaar bij >1 resultaat) — in de praktijk wil je vaak maar een paar losse albums
  toevoegen uit een grote collectie, niet alles. Zelfde interactiepatroon als Open Library dus, de
  eerdere "want dat is een hele bestaande collectie"-redenering hierboven bleek niet houdbaar.
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
  nieuwe import. ~~De MusicBrainz-zoekicoon in de reeks-kop (voor wishlist-items die je nog niet in
  bezit hebt) werkte net als bij Boeken/Strips: zoekt direct op de artiestnaam van die reeks.~~
  **Verwijderd (2026-09-13, v1.24.0)**, zie onderaan deze sectie.
- "Jaar van uitgave" (2026-09-13): extra tekstveld toegevoegd bij Boeken, Muziek én Strips (bij
  Strips naast het bestaande Volgnummer). Wordt automatisch ingevuld waar de bron het al meegeeft:
  Discogs-bulkimport (`basic_information.year`), Open Library-auteurzoeken (`first_publish_year`,
  nu ook in de fields-lijst) en de ISBN-lookup
  (`publish_date`, jaartal eruit geregexed). Bij handmatig toevoegen of via een externe bron zoals
  een Wikipedia-screenshot (zie hieronder) vult de gebruiker het zelf in. Backfill bij invoering:
  Muziek's 162 bestaande items kregen hun jaar via een herhaalde Discogs-collectie-call (146/162
  hadden een jaar; de rest ontbreekt simpelweg in Discogs' eigen data).
- ~~Discogs-collectie importeren (2026-09-13, schermeconomie): de knop staat niet meer als aparte
  altijd-zichtbare rij onder de lijstnaam, maar als icoon in de detail-header naast tandwiel/camera
  (net als bij Boeken/Strips is de globale import-rij bewust weg voor een handeling die zelden
  gebeurt) — alleen zichtbaar als er een Discogs-gebruikersnaam is ingevuld; zonder gebruikersnaam
  staat de hint ("vul je gebruikersnaam in...") gewoon op de oude plek in de lijst zelf.~~
  **Herzien (2026-09-13, v1.25.0):** het download-icoontje in de compacte header stond los van het
  "Toevoegen"-tabblad (zie de mode-tabs verderop) — precies het tabblad dat bedoeld is om alle
  manieren om iets toe te voegen bij elkaar te tonen. Wie naar "Toevoegen" ging voor Muziek zag
  alleen het handmatige formulier en moest daarnaast nog weten dat een klein icoontje bovenin de
  hele collectie importeert. Gevonden bij een grondige UX-review, niet expliciet gemeld. Nu net als
  bij Boeken (Open Library) gewoon een knop ("Discogs-collectie importeren") binnen het
  "Toevoegen"-tabblad zelf (`renderImportSection`, geen aparte discogs-specifieke uitzondering meer);
  het losse icoon in de header is vervallen.
- Stripreeksen vullen op basis van Wikipedia-lijsten (2026-09-13): geen geautomatiseerde
  in-app-scraper gebouwd (Wikipedia-pagina's zijn losse, wisselend opgemaakte tabellen — geen
  stabiele API zoals Open Library/Discogs, en zelf de juiste pagina laten raden bij een reeksnaam is
  foutgevoelig, zie ook de afgewezen AskUserQuestion hierover). In plaats daarvan: de gebruiker stuurt
  screenshots van (een deel van) zo'n Wikipedia-lijst in de chat, Claude leest de tabel visueel uit
  (nr/titel/datum) en stelt een SQL-insert voor ter goedkeuring — zelfde werkwijze als elke andere
  migratie. Op deze manier is de volledige Suske en Wiske-reeks (VK-nummering 67 t/m 385, incl.
  jaartal per album) in één keer gevuld. Belangrijk bij het gebruik van meerdere nummeringskolommen
  in zo'n screenshot (Wikipedia's chronologische Nr. vs. de nummering die letterlijk op de rug van
  de fysieke boeken staat, bv. Vierkleurenreeks bij Suske en Wiske): altijd de nummering gebruiken
  die op de fysieke uitgave staat, niet de eerste kolom klakkeloos aannemen.
- Stripcovers ophalen via stripinfo.be (2026-09-13): analoog aan de Wikipedia-lijst-aanpak, maar dan
  voor covers i.p.v. titels/jaartallen — losstaand van elkaar te gebruiken. stripinfo.be heeft geen
  publieke API, maar wel bruikbare structuur: `https://stripinfo.be/reeks/index/<reeks-id>_<Naam>`
  (vind je via `/zoek/zoek?zoekstring=...`) geeft één pagina met ALLE albums van een reeks incl.
  hun eigen strip-id in de link (`reeks/strip/<strip-id>_<Reeksnaam>_<nr>_<Titel>`) — dat is genoeg
  om albums te matchen op titel (genormaliseerd: underscores/streepjes naar spaties, diacritics eruit)
  zonder per album een aparte pagina te hoeven laden. **Val (2026-09-13): de cover-URL zelf is NIET
  simpelweg `image.php?s=<strip-id>` te raden** — dat leek in een eerste test te werken maar gaf bij
  een steekproef de cover van een compleet ANDERE strip terug. De juiste URL is
  `image.php?i=<image-id>&s=<strip-id>`, waarbij `i` alléén te vinden is op de eigen albumpagina
  van dat strip-id (`reeks/strip/<strip-id>_x` volstaat als minimale URL) — dus wél één page-fetch
  per album nodig, niet te vermijden. Gedownloade covers worden opnieuw gehost in de bestaande
  Supabase-bucket `afbeeldingen` (net als handmatige foto-uploads) — **niet** direct naar
  stripinfo.be gelinkt, want die site stuurt `Cross-Origin-Resource-Policy: same-site` mee, wat
  cross-origin `<img>`-gebruik vanaf de Lijster-site blokkeert (geverifieerd door de afbeelding
  in-browser te laden vanaf een ander origin: mislukte). Matching tussen onze albumtitels en
  stripinfo's titels is niet altijd 1-op-1: stripinfo's eigen interne volgnummering wijkt vaak af
  van de onze (andere Nederlandse vertaaledities, andere publicatievolgorde) — titel-matching (met
  een fuzzy fallback) is betrouwbaarder dan op nummer matchen. Voor Suske en Wiske (319 albums) en
  Asterix (42 albums, waarvan er 6 een net iets andere Nederlandse titel bleken te hebben op
  stripinfo — handmatig gekoppeld) is dit al succesvol gedaan; niet elk album staat op stripinfo
  (een nog niet verschenen toekomstig album, of een enkel album zonder coverscan) — dat is geen bug,
  gewoon een ontbrekend brongegeven, op te lossen met de bestaande per-item foto-upload.
- Reeksen vooraf aangemaakt zonder albums (2026-09-13, in "Strips Bert"): Asterix, Lucky Luke,
  Bollie & Billie, Yoko Tsuno, Rik Ringers, Largo Winch, De partners, Kuifje, Robbedoes, Idefix,
  Blake & Mortimer, Alex, Alex Senator, Jerom, De Rode Ridder, Thorgal (+ afgeleiden: De Jeugd van
  Thorgal, Kriss de Valnor, Louve, Wendigo), Blacksad — leeg totdat de gebruiker per reeks
  screenshots aanlevert om te vullen zoals bij Suske en Wiske/Asterix.
- Overzichtspagina/"Alle lijstjes" (2026-09-13 herzien voor schermeconomie): elk lijstje was een
  grote losse kaart (16:9-omslagfoto full-width + titel/pil/omschrijving/aantal + preview van de
  eerste 3 items) — op iPhone vulde één kaart bijna het hele scherm, dus met een paar lijstjes was
  scrollen door de kaarten zelf al vervelend. Nu een compacte rij per lijstje: klein vierkant
  omslagfotootje (52×52) links, titel + pil/aantal ernaast, verplaats/verwijder-knoppen rechts —
  geen preview-items en geen omschrijving meer (zelfde soort keuze als eerder bij de
  detailweergave-header: minder relevant dan de ruimte die het kost). `#grid` is daarmee van een
  CSS-grid met kaarten omgezet naar een simpele verticale lijst (`display:flex; flex-direction:
  column`) — een aparte multi-kolom-indeling op desktop bleek geen meerwaarde te hebben t.o.v. één
  consistente compacte lijst op elke breedte.
- ~~MusicBrainz-artiest+titel-zoeken (reeks-scoped icoon + globale "Artiest toevoegen"-knop,
  cover via Cover Art Archive) werd op 2026-09-13 nog losgekoppeld van de Discogs-Muziekbron via
  `secondarySearchSource(entry)`, en de "alles standaard aangevinkt"-bug (v1.15.2) werd nog gefixt
  (zelfde patroon als bij Open Library: resultaten standaard leeg, aanvinken = toevoegen).~~
  **Volledig verwijderd (2026-09-13, v1.24.0)** — expliciet verzoek: "Die hele Musicbrainz connectie
  mag eruit. Heeft nooit resultaat en is alleen verwarrend naast de discogs connectie, die wel goed
  werkt." Bevestigd met een screenshot van een mislukte zoekopdracht op eigen telefoon. Verwijderd:
  de globale "Artiest toevoegen"-knop, de reeks-scoped MusicBrainz-zoekicoon (Muziek-reeksen hebben
  nu geen `secondarySearchSource` meer — `secondarySearchSource(entry)` geeft alleen nog
  `'openlibrary'` terug, nooit meer `'musicbrainz'`), de "Muziekbron"-keuze in "Velden bewerken"
  (was MusicBrainz vs. Discogs — nu altijd Discogs, geen select meer nodig, direct de
  gebruikersnaam-invoer), de MusicBrainz-fetch-branches in `importSearch()`/`importPickCandidate()`
  (die functies doen nu alleen nog Open Library), en de Cover Art Archive-koppeling. Muziek-preset
  default `auto_import` gewijzigd van `'musicbrainz'` naar `'discogs'`. Bestaande Muziek-items/
  tracklist-data (allemaal al via Discogs gevuld) ondervinden geen impact.
- Cover van een item vergroot bekijken (2026-09-13): klik/tik op het kleine omslagfotootje in de
  tabel opent een lightbox (donkere overlay, sluiten via kruisje/Escape/ergens buiten de foto
  klikken) — losstaand van de normale rij-klik die het item in bewerk-modus zet
  (`e.stopPropagation()` op de thumbnail zelf).
- Bewerk-modus van een item: opslaan/annuleren stonden als kale ✓/✕-icoontjes zonder zichtbaar
  label — verwarrend, want ✕ betekent hier "annuleren", terwijl ✕ elders in de app juist
  "verwijderen" betekent (op andere plekken altijd met `.danger`-styling en een eigen aria-label,
  maar het kale icoon zelf oogt hetzelfde). Nu gewoon tekstknoppen "Opslaan"/"Annuleren", zelfde
  patroon als de andere formulieren in de app (bv. bij "+ Toevoegen").
- Tracklist per muziekalbum (2026-09-13): bron is Discogs (niet MusicBrainz) — bewuste keuze, want
  Discogs' tracklijsten horen bij de specifieke fysieke persing (kant A/B bij vinyl etc.), wat beter
  past bij een fysieke collectie dan MusicBrainz' generiekere release-group-data (die geen tracklist
  heeft zonder eerst een specifieke, vaak dubbelzinnige "release"-editie te kiezen). Bij elke
  Discogs-add wordt automatisch `GET /releases/{id}` opgehaald (tracklist zit niet in de
  bulk-collectie-call) en als "Positie. Titel (duur)" per regel opgeslagen — met een korte pauze
  tussen items bij een grotere batch, want dit is een aparte aanroep per item (Discogs' limiet is
  25/min onbevestigd, zie eerdere notitie). Bestaande 162 items met terugwerkende kracht gevuld via
  een rustig getempode achtergrond-script (zelfde aanpak als de stripcovers-backfill). In de tabel
  een in-/uitklap-icoontje (alleen zichtbaar als er een tracklist is) dat een extra rij toont; in de
  bewerk-modus van een item een eigen tekstvak (alleen bij Muziek-lijstjes, `auto_import` discogs)
  om 'm handmatig te zetten/aan te passen.
  **Aanvulling (2026-09-13, v1.17.0): los "Tracklist ophalen (Discogs)"-knopje** bij een item zonder
  tracklist (dus ook bij handmatig toegevoegde items, die nooit automatisch een tracklist bijkrijgen)
  — zoekt op Discogs' `database/search`-endpoint (bevestigd: geen auth nodig,
  wel `artist`+`release_title` als aparte parameters i.p.v. alles in één `q=` proppen, dat geeft
  veel minder ruis) op reeksnaam (artiest) + itemtitel, optioneel versmald op vorm (`format=Vinyl`/
  `CD`) als het item al eenduidig als LP XOF CD is aangevinkt. Toont een kandidatenlijst met
  titel + jaar + vorm (bv. "1981 · Vinyl, LP, Compilation") — bewust GEEN coverthumbnail in deze
  lijst, want Discogs' zoek-endpoint geeft (i.t.t. de collectie- en release-detail-endpoints) geen
  omslagfoto's mee, en per kandidaat apart een release-detail ophalen voor alleen een thumbnail is
  bij tientallen resultaten niet reëel. Pas ná het kiezen van één kandidaat wordt de volledige
  release opgehaald (`GET /releases/{id}`, dezelfde aanroep als de bulkimport al gebruikt) —
  toont dan alsnog de cover ter bevestiging, samen met de volledige tracklist, vóór je op
  "Toevoegen" klikt. Aanleiding: "best of"-compilatiealbums bestaan vaak in 5-10+ Discogs-edities
  (verschillende jaren/vormen) met soms afwijkende tracklists — silent auto-matchen op titel alleen
  zou zomaar de verkeerde tracklist kunnen opleveren, dus een expliciete kandidatenkeuze (zelfde
  patroon als de Open Library-auteurskandidaten bij Boeken) is hier bewust gehandhaafd i.p.v.
  automatisch te raden. **Fix (2026-09-13, v1.21.1): icoon van deze knop hergebruikte per ongeluk
  `tracklistSvg()`** — hetzelfde icoon als de in-/uitklap-toggle voor een tracklist die al bestaat,
  dus zonder label niet te onderscheiden (concreet gemeld: "welk knopje is dat dan? Dat is nu niet
  duidelijk"). Icoon nu `coverSearchSvg()` (vergrootglas), dezelfde die al gebruikt wordt voor de
  reeks-scoped Open Library-zoekknop — consistent "vergrootglas = extern opzoeken"-icoontaal.
  **Aanvulling (2026-09-14, v1.26.0): kandidaten gesorteerd op populariteit.** Discogs'
  `database/search` geeft per resultaat ook `community.have` mee (aantal Discogs-gebruikers dat
  precies díe persing in bezit heeft) — kandidaten staan nu aflopend op dat aantal, zodat de meest
  voorkomende/waarschijnlijke editie vanzelf bovenaan staat i.p.v. in willekeurige (Discogs-eigen)
  volgorde.
- Strips-sjabloon (2026-09-13 herzien): generieke "In bezit" vervangen door "Fysiek"/"Digitaal",
  zelfde reden als bij Boeken (e-book/fysiek) — veel strips heeft de gebruiker in digitale vorm
  (CBR/CBZ, gelezen via een externe comicreader-app, zie hieronder), dus één generiek "in bezit"
  dekte dat onderscheid niet meer. "Gelezen" blijft. Bestaande data gemigreerd: oude `in_bezit`-
  waarde → `fysiek`, `digitaal` overal op false (gebruiker vinkt zelf aan waar van toepassing).
- Items worden getoond als een tabel (`.items-table` in `.items-scroll`, `overflow-x:auto`):
  titelkolom sticky links, vinkjes-koppen (E-book/Auteur/etc.) ÉÉN keer bovenaan i.p.v. per rij
  herhaald, actieskolom (foto/bewerk/verwijder) sticky rechts. Tabelbreedte wordt expliciet in JS
  gezet (`180 + aantal_vinkjes * 54 + 90` px) i.p.v. CSS `min-width:max-content` — die laatste
  dwong ALLE tekst in de tabel tot één regel (titelkolom liep op tot 9000px), een echte val bij
  scrollbare tabellen: gebruik een berekende pixelbreedte, niet `max-content`, als je select
  kolommen vast moet houden terwijl andere mogen wrappen.
  **Fix (2026-09-13, v1.17.1): colSpan-rijen (reeks-kop, bewerk-form, cover-/tracklist-zoekpaneel)
  vielen op iPhone gedeeltelijk buiten beeld** — zo'n rij erft de volle (JS-berekende) tabelbreedte,
  die bij een paar vinkjes-kolommen (bv. Muziek: 3 vinkjes → 432px) een smalle iPhone al kan
  overschrijden; met alleen `overflow-x:auto` op de tabel bleef bv. het "Jaar van uitgave"-veld in
  het bewerkformulier daardoor onbereikbaar zonder zelf naar rechts te scrollen, zonder enige hint
  dat dat nodig was (concreet gemeld met een iPhone-screenshot: "Jaar van..." en de kolomkop "IN
  BEZIT" liepen af, niet zichtbaar dat er meer te scrollen viel). Gefixt met een gedeelde
  `.wide-row-content`-klasse: `position:sticky;left:0` houdt zo'n rij bij de zichtbare linkerrand
  (net als de bestaande sticky titel-/actieskolom), en `max-width:calc(100vw - 88px)` (viewport
  min de vaste `.wrap`+`.detail-body`-marges) laat de al aanwezige `flex-wrap` ook echt naar een
  nieuwe regel omslaan i.p.v. buiten beeld doorlopen. Dit soort bug valt niet op in de Claude
  Browser-testtool op standaardbreedte — moet je expliciet op een smalle viewport (bv. 375px)
  testen, of zoals hier: een screenshot van de gebruiker zelf.
  **Vervolg (2026-09-13, v1.21.1): `calc(100vw - 88px)` bleek op de eigenlijke iPhone van de
  gebruiker toch nog net te breed** (opnieuw gemeld met een screenshot: "Jaar van uitgave" viel
  nog steeds gedeeltelijk af) — `100vw` wordt op iOS Safari niet altijd gelijk berekend aan de
  échte zichtbare breedte (bekende platform-quirk, reproduceert niet in de Chromium-gebaseerde
  Claude Browser-testtool hier, wat het extra lastig te vinden maakte). Robuustere fix: na elke
  render (`applyWideRowContentWidths()`, aangeroepen direct na `detailCardEl.appendChild(dbody)`)
  de daadwerkelijk gemeten `clientWidth` van `.items-scroll` als inline `max-width` op elk
  `.wide-row-content`-element zetten — een echt gemeten breedte in JS is immuun voor dit soort
  CSS-eenheid-verschillen tussen browsers, de `calc(100vw - …)`-regel in CSS blijft alleen staan
  als fallback voor het allereerste beeld vóór de JS-pass. Les: voor dit soort viewport-breedte-
  aannames is een gemeten waarde (element.clientWidth) betrouwbaarder dan een CSS-eenheid als
  `vw`, zeker zodra Safari in het spel is.
  **Nóg een vervolg (2026-09-13, v1.23.0): zelfs met de gemeten breedte bleef het bewerkformulier op
  de echte iPhone van de gebruiker soms nog buiten beeld** — vermoedelijke oorzaak: `position:sticky`
  op een element BINNEN een `<td>` (i.p.v. op de `<td>` zelf) staat bekend om onbetrouwbaar gedrag
  op WebKit/iOS (in tegenstelling tot de sticky titel-/actieskolom hierboven, die sticky rechtstreeks
  op de `<td>` zelf zetten — dát werkt wel overal). Als de tabel al naar rechts gescrold was (bv. om
  de LP/CD-vinkjes te bekijken) vóórdat je op een item klikte, en sticky daar niet aansloeg, bleef
  het bewerkformulier op zijn "natuurlijke" positie in de brede tabel staan i.p.v. mee te schuiven
  naar de zichtbare linkerrand — ook al was zijn eigen breedte wél correct beperkt. Fix: `.items-
  scroll`'s scrollpositie expliciet terugzetten naar 0 (`resetItemsScroll()`) op het moment dat een
  bewerkformulier/cover-zoekpaneel/tracklist-zoekpaneel wordt GEOPEND (niet bij elke render) — zo
  hangt zichtbaarheid niet af van of `position:sticky` het in die specifieke situatie wel of niet
  doet. Dit issue was op de standaardbreedte van de Claude Browser-testtool hier niet te reproduceren
  (bevestigd via `getBoundingClientRect()`-metingen: daar klopte alles al) — pas na herhaalde
  screenshots van de gebruiker zelf, met de tabel al gescrold, kwam de werkelijke oorzaak boven
  water. Les: bij hardnekkige "valt buiten beeld"-meldingen die niet reproduceren, vraag naar (of
  test zelf) de exacte voorafgaande staat (was er al gescrold?) i.p.v. alleen de eindsituatie te
  meten.
  **Gat gedicht (2026-09-13, v1.25.0):** `resetItemsScroll()` werd bij de invoering hierboven op 4
  plekken aangeroepen, maar niet in `openImportFlow()` (het reeks-scoped Open Library-zoekpaneel en
  de Discogs-collectie-import — óók een wide-row-content-rij). Gevonden bij een grondige UX-review
  (niet zelf reproduceerbaar in de Claude Browser-testtool, waar `scrollLeft` bij een nieuw gerenderd
  element altijd al op 0 staat) maar wel een structurele inconsistentie t.o.v. het hierboven
  vastgelegde patroon — nu ook daar toegevoegd, voor de zekerheid.
  **Aanvulling (2026-09-13, v1.20.0): vinkjes ook in het toevoeg- en bewerkformulier** — de
  vinkjes-kolommen zelf zijn nog steeds niet sticky (bewuste keuze, zie hierboven: alleen titel-/
  actieskolom zijn sticky, de vinkjeskolommen mogen gewoon meescrollen), maar bleken daardoor op
  iPhone praktisch onbereikbaar: geen zichtbare hint dat er meer kolommen waren om naar toe te
  scrollen. Concreet gemeld: "ik kan nergens het medium (CD/vinyl) zien of aanpassen, niet op
  iPhone". Losse checkbox-set (`buildVinkjesFieldset()`, hergebruikt `.checkbox`-stijl uit de
  tabel) nu ook in: het toevoegformulier (nieuw item meteen met het juiste vinkje aanmaken i.p.v.
  altijd ongevinkt beginnen — aanleiding: "is het niet logisch om bij toevoegen van muziek het
  medium gelijk aan te geven?") én het item-bewerkformulier (bestaand item aanpassen zonder de
  tabel te hoeven scrollen). Generiek op basis van `vink_velden`, dus geldt voor elk lijsttype
  (Boeken: E-book/Fysiek/Gelezen, Strips: Fysiek/Digitaal/Gelezen, niet alleen Muziek LP/CD).
  **Achterliggende bug pas nu écht gevonden en gefixt (2026-09-13, v1.25.0): de vinkjeskolommen in de
  tabel zélf waren op iPhone-breedte volledig onbereikbaar, hoe ver je ook scrolde.** De bovenstaande
  toevoeging aan het toevoeg-/bewerkformulier was destijds een (nuttige, maar gedeeltelijke) omweg —
  het achterliggende probleem in de tabel zelf bleef onopgemerkt tot een grondige UX-review met
  metingen in devtools. Oorzaak: de sticky titelkolom (~160px) en de sticky actiekolom (~90-122px,
  groeit mee met het aantal icoontjes) zijn SAMEN al bijna of helemaal net zo breed als een
  telefoonscherm — gemeten op 375px: 160+122=282px sticky, nog maar 3px "vrij" op 285px binnenwerk.
  De vinkjeskolommen (niet sticky, bedoeld om te scrollen) zaten daardoor bij elke scrollpositie
  volledig verstopt ÓNDER een van beide sticky kolommen (ondoorzichtige achtergrond, hogere
  z-index) — optisch niet te onderscheiden van "gewoon leeg", dus nooit eerder als bug herkend.
  Bevestigd met `getBoundingClientRect()`-metingen en een tijdelijke debug-outline vóór de fix.
  ~~Fix: op smalle schermen (`@media (max-width: 480px)`) is de actiekolom niet meer sticky (`position:
  static`) — daardoor is er nog maar ÉÉN sticky kolom (titel) en werkt scrollen weer normaal om de
  vinkjes te bereiken; bewerken blijft zonder scrollen mogelijk via een tik op de titel zelf (dat
  deed al hetzelfde als het potlood-icoontje). Op desktop-breedte ongewijzigd (daar was al genoeg
  ruimte, geen media-query nodig).~~
  **Vervangen (2026-09-14, v1.26.0, buiten deze sessie om doorgevoerd — zie onderaan dit hoofdstuk
  "Let op: sessies kunnen elkaars werk inhalen"): geen media-query meer, acties zitten nu gewoon IN
  de sticky titelkolom.** `.col-actions` (de eigen sticky kolom rechts) is volledig vervallen —
  foto/bewerken/verwijderen/tracklist-ophalen staan nu onder de titel binnen dezelfde `<td
  class="col-title">` (`colCount` dus `1 + vinkVelden.length` i.p.v. `2 + ...`). Reden: ook mét de
  v1.25.0-fix moest je voor foto/verwijderen/tracklist nog steeds naar rechts scrollen (alleen
  bewerken was al zonder scrollen bereikbaar, via een tik op de titel) — `position:sticky` zorgt er
  namelijk alleen voor dat een kolom in beeld BLIJFT terwijl je scrolt, niet dat hij al zichtbaar is
  vanaf scrollLeft:0 (de openingsstand van elk lijstje). Met alles in de titelkolom is dat hele
  onderscheid weg: alle item-acties zijn nu altijd direct bereikbaar, alleen de vinkjeskolommen
  (aanvinken, minder kritiek) blijven in het scrollbare midden. `.col-title`'s `max-width` ging van
  280px naar 220px (compenseert de extra hoogte van de acties eronder). De v1.25.0-media-query-fix
  hierboven werkte, maar deze aanpak is eenvoudiger én lost een net iets breder probleem op.
  **Herzien (2026-09-13, v1.25.0): "zachte landing" verving `scrollIntoView` door een bevestiging
  ter plekke.** ~~Na een succesvolle toevoeging via het toevoegformulier (genest én niet-genest)
  scrollt de nieuwe rij automatisch in beeld (`scrollIntoView({block:'center'})`)~~ — dit botste met
  achter elkaar meerdere items toevoegen (concreet gemeld: bij het toevoegen van 2 lp's moest je na
  elke toevoeging weer helemaal terugscrollen naar het formulier bovenaan om de volgende in te
  voeren). Nu blijft de scrollpositie gewoon bij het formulier: een korte, vanzelf verdwijnende
  bevestigingstekst (`showAddConfirmation()`, bv. `"Sheet Music" toegevoegd bij 10cc.`) verschijnt
  vlak onder het formulier (state-gedreven met een `setTimeout`, want de hele form wordt bij elke
  render opnieuw opgebouwd — een direct in de DOM geplakt regeltje zou meteen weer verdwijnen), en
  het Titel-veld krijgt weer focus voor de volgende invoer. De rij zelf licht nog steeds kort op
  (`flashItemRow()`, hernoemd van `scrollToAndHighlightItem`) als 'm toevallig al in beeld is, maar
  zonder de pagina te verslepen. **Let op**: de losse "Tracklist ophalen (Discogs)"-knop (zie
  hierboven) is bewust een aparte, expliciete actie — het toevoegformulier zelf doet GEEN
  automatische Discogs-aanroep; wie na een handmatige toevoeging ook cover/tracklist wil, moet dat
  knopje zelf gebruiken.
- **UX-herziening na kritische review + vergelijking met VinylBox (2026-09-14, v1.27.0)**: de
  eigenaar ervoer de Muziek-collectie als "moeizaam en niet intuïtief" — vroeg om een kritische blik
  en inspiratie uit VinylBox (een vergelijkbare, commerciëlere vinyl/cd-collectie-app met Discogs-
  koppeling). Kernprobleem: Lijster voelde als een spreadsheet, niet als "een platenkast
  doorbladeren" — vooral door piepkleine covers (30×42px) en 4 permanent zichtbare actie-icoontjes
  per rij (verergerd door de v1.26.0-wijziging hierboven, die ze als eigen rij ónder de titel zette).
  Bewust NIET overgenomen van VinylBox: scannen, marktplaats/verkoop, prijsinzicht — niet relevant
  voor een gedeelde huishoud-app zonder account-scheiding. Vier generieke verbeteringen doorgevoerd
  (raken dus ook Boeken/Strips/Bordspellen, niet alleen Muziek — geen type-specifieke hacks, zelfde
  filosofie als de rest van de app):
  1. **Rij vereenvoudigd**: cover van 30×42 naar 44×62px (`.item-thumb`), en van de 4 permanente
     actie-icoontjes (foto/tracklist-ophalen/bewerk/verwijder) blijft alleen de tracklist-
     bekijken-toggle over (en alleen als er al een tracklist is — puur "bekijken", geen bewerking).
     Foto wijzigen/toevoegen, cover ophalen (Open Library), tracklist ophalen (Discogs) en
     verwijderen zijn verplaatst naar het bewerkformulier zelf (tik op de titel, zoals al kon) —
     nieuwe knoppen daar: `+ Foto toevoegen`/`Wijzig foto` (met bijbehorende hidden file-input,
     zelfde `uploadItemCover()`), `Cover ophalen (Open Library)` (alleen zichtbaar zonder cover, bij
     `auto_import: 'openlibrary'`), `Tracklist ophalen (Discogs)` (alleen zichtbaar zonder tracklist,
     bij `auto_import: 'discogs'` — roept `openTrackSearch()` aan, die nu ook `state.editingItem =
     null` zet zodat de trackSearch-UI daadwerkelijk verschijnt i.p.v. het bewerkformulier open te
     houden), en een rode `Verwijderen`-tekstknop naast Opslaan/Annuleren (zelfde `delete`-call als
     de vroegere ✕-icoon, ook zonder bevestigingsvraag — dat was al zo). Schikt zelfde principe toe
     als eerder al bij de discogsBtn/downloadSvg-opruiming: minder altijd-zichtbare chrome voor
     acties die je zelden per item gebruikt.
  2. **Kaartjes i.p.v. platte teksregel bovenaan** (`.stat-cards`): i.p.v. "170/170 in bezit · 149/170
     lp · 22/170 cd" als lopende tekst, nu losse kaartjes met een groot getal + label — plus een
     nieuw "items"-kaartje (totaal aantal) en, alleen bij `entry.genest`, een "reeksen"-kaartje
     (aantal artiesten/auteurs/series). Puur visuele herschikking van bestaande data (`vinkCounts()`),
     geen nieuwe queries.
  3. **A-Z-sprongbalk** (`buildAzBar()`, `.az-bar`): bij een alfabetisch gesorteerde genest-lijst met
     meer dan 8 reeksen verschijnt een horizontale, scrollbare rij met de daadwerkelijk aanwezige
     beginletters (na `sortNaam()`, dus "The Beatles" telt als "B") — tikken op een letter scrollt
     naar de eerste reeks die ermee begint (`reeks-row`'s `data-reeks-id`, nieuw toegevoegd zodat
     'ie te targeten is). Verborgen tijdens een actieve zoekopdracht/filter (de lijst is dan al kort,
     en de beschikbare letters zouden niet meer overeenkomen met wat nog zichtbaar is). Vergelijkbaar
     met de alfabet-index in Contacten-apps; bij Muziek (95 reeksen) een groot verschil t.o.v. blind
     scrollen.
  4. **Filters als losse chips i.p.v. een Alles/Wel/Niet-segment** (`.filter-chip`): elk vinkje is nu
     één tikbare pil die door de status cyclet (alle → wel → niet → alle), met kleur als signaal
     (neutraal/groen met ✓/rood met ✕) i.p.v. drie aparte knopjes per vinkje binnen een groepje.
     Compacter (meer filters passen op één regel) en visueel dichter bij VinylBox' chip-filters. Het
     in-/uitklappen van het hele filterblok (`state.showFilters`) is ongewijzigd gebleven — alleen de
     knoppen ERIN zijn anders.
  Bewust NIET gebouwd (grotere stap, expliciet afgeraden totdat er concrete behoefte aan blijkt): een
  volledig los, navigeerbaar detailscherm per album zoals VinylBox — dat is een fundamenteel andere
  navigatievorm dan de rest van de app (inline bewerken in de tabelrij), geen kleine aanpassing.
  **Vervolg (2026-09-14, v1.27.1): covers nog groter** — expliciet gevraagd na het zien van de
  v1.27.0-wijzigingen, "zeker met de letterbalk" (de A-Z-sprongbalk maakt bladeren al makkelijker,
  grotere covers maken het bladeren zelf ook prettiger). `.item-thumb` van 44×62 naar 58×82px,
  `.col-title`'s `max-width` van 220 naar 240px mee opgehoogd zodat de titel niet extra hoeft te
  wrappen door de bredere cover.
- **Lijstje verwijderen alleen nog op de hoofdpagina (2026-09-13, v1.18.0)**: het rode "Verwijder
  lijstje"-linkje onderaan de detailweergave is verwijderd — de hoofdpagina heeft per lijstje al
  een ✕-knop met dezelfde bevestigingsvraag (`deleteList()`, "kan niet ongedaan gemaakt worden"),
  dus dit was een pure duplicaat op een plek die zelf minder voor de hand lag. Aanleiding: expliciet
  gemeld dat verwijderen intuïtief bij het overzicht hoort, niet in de detailweergave van het
  lijstje zelf.
- **Item bewerk-form: cover-thumbnail + grotere tracklist (2026-09-13, v1.18.0)**: het inline
  bewerkformulier van een item toonde geen omslagfoto (wel aanwezig in de gewone rij-weergave via
  `.item-thumb`) — nu ook een kleine (52×52) thumbnail vooraan in het formulier als het item een
  cover heeft, met dezelfde klik-voor-lightbox-interactie. Tracklist-tekstvak ging van
  `min-height:70px` (~3 zichtbare regels, te weinig voor een LP-kant met 8-12 tracks) naar 200px. (logo/tagline/"Nieuw lijstje") verdwijnt in de lijst-detailweergave (alleen "← Alle
  lijstjes" + lijstnaam) — was op iPhone te veel verloren ruimte vóór je daadwerkelijk items ziet.
  Lege omslagfoto-placeholder in detailweergave vervangen door een smal "+ Omslagfoto toevoegen"-
  linkje i.p.v. een grote lege blok van 140px+.
- Zoeken (op titel + alle tekstvelden) en sorteren (Handmatig/Titel/per tekstveld) per lijstje,
  boven de items. Bij een actieve sortering verdwijnen de handmatige verplaats-pijltjes. Het
  zoekveld heeft een eigen "x"-knopje om de tekst te wissen (2026-09-13, v1.16.1) i.p.v. te
  vertrouwen op de native browser-clearknop van `type="search"` — die is inconsistent aanwezig
  tussen browsers/platforms (o.a. onopvallend/afwezig op iPhone), dus een eigen zichtbare knop
  is betrouwbaarder.
  **Aanvulling (2026-09-13, v1.22.0): sorteren op "Artiest/auteur/reeks"** bij `genest` lijstjes
  (Boeken/Muziek/Strips) — herordent de REEKSEN zelf alfabetisch (`localeCompare` met 'nl'-locale),
  i.p.v. alleen de items binnen een reeks te sorteren (dat deed "Titel" al). Zonder deze optie
  stonden reeksen altijd in handmatige/toevoegvolgorde, dus een net toegevoegde artiest (bv.
  Creedence Clearwater Revival) belandde altijd onderaan i.p.v. bij de C's. Zelfde patroon als bij
  item-sortering: bij een actieve alfabetische reeks-sortering verdwijnen de handmatige reeks-
  verplaats-pijltjes (omhoog/omlaag) uit de reeks-kop, want handmatig verslepen heeft geen zin
  zolang de volgorde toch alfabetisch herberekend wordt.
  **Aanvulling (2026-09-13, v1.25.1): lidwoord-voorvoegsel "The" wordt genegeerd bij het alfabetisch
  sorteren van reeksen** — explicit gevraagd: "The Beatles" hoort bij de B, niet bij de T (zelfde
  conventie als een platenkast/bibliotheek). `sortNaam(naam)` strip alleen een leidend `"the "`
  (case-insensitive, Engelse bandnamen) vóór de `localeCompare`. Bewust GEEN Nederlandse lidwoorden
  ("De"/"Het") gestript: die kunnen ook gewoon onderdeel zijn van een eigennaam/achternaam (bv. de
  strip-reeks "De Kiekeboes", of een auteursnaam) en zouden dan onterecht verplaatst worden — de
  scope is expliciet gehouden tot het gevraagde "The"-geval.
  **Herzien (2026-09-13, v1.23.0): alfabetisch op reeks is nu de DEFAULT** bij elk genest lijstje
  (`defaultSortKey(entry)`: 'reeks' als `entry.genest`, anders 'volgorde') — expliciet gevraagd:
  "doe de alfabetische sortering als default, bij alle lijstjes". `state.sortBy` is per-sessie
  (niet persistent), dus dit is puur een gewijzigde fallback-waarde als de gebruiker zelf nog niets
  koos; "Handmatig" blijft gewoon een keuze in de Sorteer-select voor wie alsnog handmatig wil
  verslepen. **Strips-specifieke uitzondering**: staat er een "Volgnummer"-tekstveld op de lijst
  (`hasVolgnummerField(entry)`, generiek op veldnaam i.p.v. hardcoded op lijsttype — zelfde
  velden-systeem-filosofie als de rest van de app), dan worden items BINNEN elke reeks altíjd op
  dat veld gesorteerd (`sortByVolgnummer()`, natuurlijke/numerieke vergelijking zodat "2" vóór "10"
  komt), ongeacht welke Sorteer-optie er verder gekozen is — expliciet gevraagd: "bij strips geldt
  daarbinnen altijd: op nummer, indien aanwezig". Dit gebeurt op het platte, al gefilterde/gesorteerde
  array in `filteredItems()` (ná de gewone titel/veld-sortering, vóór het per-reeks groeperen in
  `renderNestedItems`), dus de leesvolgorde binnen een reeks (bv. Suske en Wiske 67, 68, 69…) blijft
  altijd kloppen ook al staat de Sorteer-select op iets anders zoals "Titel".
  **Herzien (2026-09-13, v1.19.0→v1.19.1): Bekijken/Toevoegen als aparte modus-tabs** i.p.v. zoeken
  + sorteren/filteren + toevoegen allemaal tegelijk boven de lijst. Expliciete aanleiding (letterlijke
  bewoording van de gebruiker): "je bent aan het zoeken, aan het bekijken (met filteren), of aan het
  toevoegen. Niet alle drie tegelijk. Maar je moet altijd makkelijk terug kunnen naar die keuze" —
  de knoppen "+ Toevoegen"/"Artiest toevoegen" pal onder de zoekbalk suggereerden een relatie die er
  niet was ("als ik zoek ben ik niet aan het toevoegen"). v1.19.0 begon met DRIE tabs (Bekijken/
  Zoeken/Toevoegen los), maar meteen daarna kwam de nuance dat zoeken en bekijken/filteren juist
  dicht bij elkaar liggen ("dat ligt misschien toch wel dichtbij elkaar. Maar toevoegen is echt een
  aparte actie") — dus in v1.19.1 teruggebracht naar TWEE tabs: `state.viewMode[lijstId]`
  ('bekijken', default, of 'toevoegen') bepaalt nu welk controle-blok boven de itemlijst staat.
  Bekijken toont zoekveld + Sorteer + Filters-knop samen (zoals het al was, vóór dit hele
  hoofdstuk), Toevoegen toont het toevoegformulier/ISBN-knop/Discogs-collectie-import (de "Artiest-
  toevoegen"-knop hoorde hier ook bij, maar is inmiddels verwijderd — zie MusicBrainz-verwijdering
  hieronder). De itemlijst zelf blijft in beide tabs gewoon zichtbaar (alleen de besturing erboven
  wisselt) — reeks-scoped zoekicoontjes in de tabel (Open Library per reeks) zijn bewust NIET aan
  de tabs gekoppeld, want die zijn al lokaal aan hun reeks-rij gebonden, geen onderdeel van de
  globale besturingsbalk waar de klacht over ging. Verlaten van Toevoegen ruimt een evt. openstaande
  ISBN-flow expliciet op. De losse "+ Toevoegen"-toggle binnen de
  genest-add-form is vervallen (het formulier staat nu meteen open zodra je op de Toevoegen-tab
  klikt, dat IS al de bewuste keuze) — na een succesvolle toevoeging leegt het formulier zichzelf
  voor de volgende (blijft in Toevoegen-modus voor snel achter elkaar meerdere items invoeren).
  **Fix (2026-09-13, v1.25.0): lege-lijst-tekst in "Bekijken" verwees naar een knop die daar niet
  stond.** Een nieuwe, lege geneste lijst toonde altijd "Nog niets toegevoegd. Gebruik de knop
  hieronder om te beginnen" — maar het toevoegformulier staat alleen boven de tabel als je al op het
  tabblad "Toevoegen" staat; in "Bekijken" bestond zo'n knop niet. Gevonden bij een grondige
  UX-review (bevestigd door een leeg testlijstje aan te maken). Tekst is nu afhankelijk van het
  actieve tabblad: in "Bekijken" wijst 'm naar het tabblad "Toevoegen" hierboven, in "Toevoegen"
  verwijst 'm naar het formulier dat er al staat. Zelfde aanpassing bij niet-geneste lijstjes
  (Lijst/Bordspellen/Aangepast).
- Volgorde wordt bijgehouden als timestamp (nieuw item/lijst = `Date.now()`); verplaatsen wisselt de
  `volgorde`-waarde van twee buren om (last-writer-wins, geen transacties nodig op deze schaal).

## Sync & offline
- Live sync via Supabase Realtime (`postgres_changes` op beide tabellen, wildcard event) — bij elke
  wijziging wordt de hele dataset opnieuw opgehaald (`refetchAll`), geen incrementele diff-logica.
  Werkt prima op deze schaal (huishoudelijke lijstjes, geen duizenden rijen).
- Service worker (`service-worker.js`) cachet alleen de app-shell (HTML/manifest/icons/supabase-js),
  read-only fallback bij geen verbinding — geen queue/sync-logica voor schrijfacties zoals bij Gezin.
  Schrijven zonder verbinding faalt gewoon met een alert; dat is bewust simpel gehouden.
- **Versie blijft hangen op iPhone (2026-09-13, gefixt v1.15.1)**: de HTML zelf wordt altijd vers
  van het netwerk gehaald (`fetch(request, {cache:'no-store'})` in de SW), maar de service worker
  zélf checkte nergens actief op updates — de browser-eigen updatecheck gebeurt volgens spec
  hooguit eens per zoveel tijd, en in een iOS-standalone-PWA (zelden echt afgesloten, geen tab-
  reload) duurt dat soms erg lang. Fix: expliciete `registration.update()` bij laden, elk uur, en
  bij terugkeer naar het scherm (`visibilitychange`), plus een `controllerchange`-listener die de
  pagina automatisch herlaadt zodra een nieuwe SW het overneemt. Omdat de HTML zelf al vers wordt
  opgehaald (zie boven), draait deze nieuwe check-logica ook al binnen een oude/vastzittende SW —
  geen kip-ei-probleem. Bij een al vastzittend toestel: één keer de PWA volledig sluiten (via de
  appswitcher, niet alleen naar de achtergrond) en heropenen met een actieve verbinding is genoeg
  om zichzelf te herstellen.

## Platform
PWA via `manifest.json` + `apple-touch-icon.png` + service worker — "Zet op beginscherm" op iPhone
geeft een fullscreen appicoon zonder Safari-balk. Geen Claude-login nodig, geen native app.

## Bekende openstaande punten (geen GO — pas oppakken na expliciete instructie)
- CBR/CBZ-strips in-app lezen: overwogen (2026-09-13) en bewust NIET gebouwd. Technisch mogelijk
  (RAR-extractie client-side kan via WASM-bibliotheken, CBZ/ZIP zou een stuk eenvoudiger zijn), maar
  een forse klus: opslagomvang (scans al snel 50-300MB per album, onduidelijk of het Supabase-
  abonnement dat aankan), grote-bestanden-upload vanaf de browser, en geheugenrisico bij het
  uitpakken in Safari op iPhone (PWA-geheugenlimieten). Gebruiker leest digitale strips liever
  gewoon in zijn bestaande comicreader-app — Lijster hoeft dat niet over te nemen, alleen het
  fysiek/digitaal-onderscheid bijhouden (zie Strips-sjabloon in Datamodel hierboven).
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
- ~~MusicBrainz-auto-import (Muziek) kon vanuit deze dev-omgeving niet betrouwbaar getest worden
  ("server is busy"-responses)~~ — bleek ook vanaf de telefoon van de gebruiker onbetrouwbaar/zonder
  resultaat, en is daarom 2026-09-13 (v1.24.0) volledig verwijderd i.p.v. verder uitgezocht.
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
  DB-kolom `lijsten.discogs_username` toegevoegd. Was aanvankelijk een los te kiezen Muziekbron
  naast `musicbrainz` (schakelaar in "Velden bewerken"); sinds de MusicBrainz-verwijdering
  (2026-09-13, v1.24.0) is `auto_import: 'discogs'` gewoon de vaste/enige Muziekbron en de
  preset-default.
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
