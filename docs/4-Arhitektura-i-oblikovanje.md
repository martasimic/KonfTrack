# 4. Arhitektura i oblikovanje

**Cilj:** Definirati arhitekturni stil, dekompoziciju programa na podsustave i module, organizaciju podataka te dinamičko ponašanje tijekom izvođenja. Arhitektura predstavlja temelj za implementaciju i ažurira se pri svakoj važnijoj promjeni projektnih odluka.

| Arhitekturni pogled | Ključno inženjersko pitanje | Primijenjeni model ili dijagram |
| :--- | :--- | :--- |
| **Strategija rješenja** | Koji stil i temeljne odluke određuju program? | Tablica odluka (odjeljak 4.1) |
| **Struktura podsustava** | Koji su glavni dijelovi i kako komuniciraju? | Dijagram komponenata |
| **Model podataka** | Kako su strukturirani trajni podaci programa? | Relacijski ER dijagram |
| **Struktura koda** | Kako su dodijeljene odgovornosti i ključne klase? | Reprezentativni dijagram razreda |
| **Dinamičko ponašanje** | Kako moduli surađuju pri obradi ključnih transakcija? | sekvencijski dijagram |
| **Stanja sustava** | Kako ključni domenski objekt mijenja svoja stanja? | Dijagram stanja |
| **Fizički razmještaj** | Na kojim se izvršnim čvorovima program izvodi? | Dijagram razmještaja (poglavlje 6) |

---

Za svaki UML dijagram čuvajte čitljivu sliku i PlantUML izvor, kako je opisano na stranici [Home](./Home.md). U dovršenoj stranici koristite vlastite projektne odluke i modele.

## 4.1 Ključne arhitekturne odluke

**Cilj:** Objasniti nekoliko arhitekturnih odluka koje najviše utječu na rješenje i navesti njihove projektno specifične razloge.

| Odluka | Razmatrana alternativa | Projektno specifično obrazloženje i kompromis |
| --- | --- | --- |
| Jedna poslužiteljska aplikacija (Node.js/Express) podijeljena na module: rute, kontrolere, servise i modele | Mikrousluge, npr. zasebne usluge za konferencije, radove i recenzije | Konferencije, radovi, recenzije i odluke međusobno su čvrsto povezani: odluka o radu ovisi o recenzijama, a recenzija o verziji rada i rokovima konferencije. Jedna aplikacija dijeli bazu i provjeru ovlasti, pa se te veze ne moraju usklađivati preko mreže, a postavlja se kao jedna usluga. Kompromis: pojedini dijelovi ne mogu se zasebno skalirati, a kvar aplikacije zaustavlja cijeli sustav. |
| Stranice se generiraju na poslužitelju iz EJS predložaka | Zaseban klijent u pregledniku (SPA, npr. React) koji komunicira s REST API-jem | Sučelje se uglavnom sastoji od obrazaca i popisa (predaja rada, pregled radova, unos recenzije), za što je poslužiteljsko generiranje dovoljno. Ista aplikacija poslužuje stranice i obrađuje obrasce, pa se sesija prenosi kolačićem bez dodatne konfiguracije CORS-a. Kompromis: slabija interaktivnost i ponovno učitavanje stranice nakon slanja obrasca, a frontend i backend moraju se uskladiti oko naziva polja i varijabli u predlošcima. |
| Relacijska baza podataka (PostgreSQL) | Dokumentna baza podataka (npr. MongoDB) | Podaci su izrazito povezani: autori i recenzenti vezani su uz radove, recenzije uz verzije radova, a radovi uz konferencije i sekcije. Strani ključevi sprječavaju, primjerice, recenziju nepostojećeg rada, a rokovi se pohranjuju s vremenskom zonom (`timestamptz`) za usporedbu s trenutkom predaje. Kompromis: svaka promjena sheme zahtijeva migraciju. |
| Pristup bazi preko Prisma ORM-a s migracijama u repozitoriju | Ručno pisani SQL upiti i skripte | Na poslužiteljskom dijelu radi više članova tima, a shema u `schema.prisma` i migracije u repozitoriju osiguravaju da svi rade nad istom strukturom baze. Kompromis: shema je preuzeta iz postojeće SQL baze pa neki nazivi relacija nisu čitljivi, a ograničenja tipa `CHECK` zahtijevaju dodatnu ručnu obradu u migracijama. |
| Sesija s `httpOnly` kolačićem za web-stranice i JWT (Bearer) za API | Samo sesije ili samo JWT | Web-stranice koriste sesiju koja se regenerira pri prijavi i uništava pri odjavi. API rute (`/api/...`) prihvaćaju i JWT u zaglavlju, što omogućuje ispitivanje API-ja bez preglednika. Isti middleware prihvaća oba načina. Kompromis: održavaju se dva mehanizma, a izdani JWT ostaje valjan do isteka (24 sata) i nakon odjave. |
| Prijava putem ORCID-a (OAuth 2.0, *authorization code*) uz prijavu e-poštom i lozinkom | Samo prijava e-poštom i lozinkom | Korisnici su znanstvenici koji uglavnom već imaju ORCID iD. Prijava ORCID-om pouzdano povezuje račun s identitetom istraživača, što je preduvjet za provjeru sukoba interesa na temelju koautorstva. Kompromis: ovisnost o vanjskoj usluzi i odvojene postavke za sandbox i produkcijsko okruženje; korisnik koji se prijavljuje samo ORCID-om nema lozinku pa token dobiva isključivo kroz web-prijavu. |
| PDF datoteke radova pohranjuju se u datotečni sustav poslužitelja, a baza čuva putanju; svaka predaja stvara novu verziju rada (`paper_versions`) | Pohrana PDF-a u bazi (`bytea`) ili u objektnoj pohrani (npr. S3) | Baza ostaje mala, a datoteke nisu javno dostupne jer se mapa `uploads/` ne poslužuje kao statički sadržaj. Pristup datoteci zato mora proći provjeru ovlasti na poslužitelju. Recenzija je vezana uz konkretnu verziju, pa nakon revizije ostaje jasno koju je verziju recenzent ocijenio. Kompromis: datoteke se ne nalaze u sigurnosnoj kopiji baze, a na platformi s privremenim datotečnim sustavom mogu se izgubiti pri ponovnom postavljanju (vidi 4.7). |
| Anonimnost dvostruko slijepe recenzije provodi poslužitelj | Skrivanje podataka o autorima samo u korisničkom sučelju | Ako se podaci o autorima samo skrivaju u sučelju, i dalje se mogu pročitati iz API odgovora, naziva datoteke ili metapodataka PDF-a. Zato poslužitelj recenzentu ne vraća podatke o autorima, datoteke se spremaju pod generiranim nazivima, a pri predaji se provjerava sadrže li metapodaci PDF-a ime autora. Kompromis: svaki upit koji vraća podatke recenzentu mora se zasebno filtrirati, a provjera metapodataka samo upozorava i ne može otkriti ime autora u samom tekstu rada. |

Objasnite glavne arhitekturne odluke bez ponavljanja popisa tehnologija. Ako važna odluka ima ADR, dovoljno je u tablici navesti njegov ID.

<details>
<summary><strong>Primjer arhitekturnih odluka i kompromisa</strong></summary>

**Primjer — CrisisMap:**

| Odluka | Razmatrana alternativa | Projektno specifično obrazloženje i kompromis |
| --- | --- | --- |
| Klijent u pregledniku i jedan poslužiteljski API | Više zasebno postavljenih poslužiteljskih usluga | Klijent prikazuje kartu, dok poslužiteljski sustav provjerava i pohranjuje prijave. Jedan poslužiteljski sustav upravljiv je za ovaj opseg; njegov kvar može privremeno prekinuti rad s prijavama. Obrazloženje je u ADR-01. |
| Relacijska pohrana prijava | Dokumentno orijentirana pohrana | Prijave imaju prepoznatljive veze s korisnicima i lokacijama. Relacijska shema te veze čini eksplicitnima, ali zahtijeva migracije kada se shema promijeni. |

**Analiza primjera:** Obje odluke navode razlog vezan uz aplikaciju za prijavljivanje kriznih događaja i priznaju kompromis. Odluka o poslužiteljskom sustavu upućuje na ADR-01 umjesto ponavljanja cijele usporedbe. Moduli unutar jedne aplikacije koja se postavlja kao cjelina nisu zasebne mikrousluge. U vlastitoj tablici navedite alternative koje je tim doista razmatrao.

</details>

## 4.2 Gradivni blokovi i UML dijagram komponenata

**Cilj:** Prikazati logičku podjelu programa na komponente, njihova sučelja i vanjske integracije. Dijagram komponenata prikazuje logičku organizaciju, dok dijagram razmještaja prikazuje fizičko izvođenje. Ovaj prikaz ažurirajte kako sustav poprima konačan oblik.

**Dijagram komponenata:** [Ugradite čitljiv UML dijagram komponenata.]  
**PlantUML izvor:** [Poveznica na odgovarajuću `.puml` datoteku.]

| Komponenta | Odgovornost | Razlog za ovu granicu |
| --- | --- | --- |
| **Web-klijent (EJS predlošci)** | Generira korisničko sučelje za prijavu, registraciju, predaju radova, pregled recenzija i javni program konferencije te šalje HTTP zahtjeve poslužitelju. | Korisnička interakcija i prikaz podataka odvojeni su od poslovne logike i trajne pohrane podataka. |
| **HTTP sloj (Express rute i kontroleri)** | Obrađuje HTTP zahtjeve, provjerava sesiju ili JWT token, provodi autorizaciju i usmjerava zahtjeve prema odgovarajućim servisima. | Komunikacija preko HTTP-a i sigurnosne provjere izdvojene su od domenskih pravila sustava. |
| **Domenski servisi i poslovna logika** | Implementiraju pravila predaje radova, upravljanje konferencijama i recenzijama, anonimizaciju podataka za dvostruko slijepu recenziju te koordiniraju provjeru PDF dokumenata i DOI oznaka. | Poslovna pravila izdvojena su iz kontrolera radi jednostavnijeg održavanja, ponovne uporabe i testiranja. |
| **PdfService** | Provjerava format PDF datoteka, metapodatke dokumenta i potencijalne podatke koji bi mogli otkriti identitet autora. | Funkcionalnost validacije i anonimizacije dokumenata koristi se na više mjesta te je izdvojena u zaseban servis. |
| **CrossrefService** | Komunicira s Crossref API-jem radi provjere valjanosti DOI oznaka navedenih u radu. | Integracija s vanjskim sustavom centralizirana je na jednom mjestu kako bi ostatak aplikacije bio neovisan o detaljima komunikacije. |
| **Prisma ORM** | Omogućuje pristup relacijskoj bazi podataka, upravlja relacijama među entitetima i migracijama sheme baze. | Pristup podacima izoliran je od poslovne logike kako bi se smanjila izravna ovisnost o SQL upitima. |
| **PostgreSQL baza podataka** | Trajno pohranjuje korisnike, konferencije, teme, radove, verzije radova, autore, recenzije i ostale povezane podatke. | Relacijska baza omogućuje očuvanje integriteta međusobno povezanih podataka putem stranih ključeva i ograničenja. |
| **Datotečni sustav poslužitelja (`uploads/`)** | Pohranjuje PDF verzije predanih radova pod generiranim nazivima datoteka. | Binarne datoteke odvojene su od baze podataka kako bi se smanjila njezina veličina i omogućila jednostavnija obrada dokumenata. |
| **ORCID API** | Omogućuje autentikaciju korisnika putem OAuth 2.0 protokola te dohvat ORCID identifikatora istraživača. | Vanjski sustav pruža potvrdu identiteta istraživača bez potrebe za implementacijom dodatnog mehanizma provjere. |
Sustav **KonfTrack** implementiran je kao jedna web-aplikacija podijeljena na logičke komponente različitih odgovornosti. Korisničko sučelje generira se na poslužitelju, dok se poslovna pravila, pristup podacima i integracije s vanjskim sustavima nalaze u odvojenim modulima. Vanjske usluge ORCID i Crossref nalaze se izvan granice sustava te su dostupne kroz jasno definirana sučelja.
Započnite crtanjem granice oko predloženog sustava. Unutar nje prepoznajte dijelove s različitim odgovornostima; planiranu bazu podataka aplikacije smjestite unutar granice, a usluge trećih strana izvan nje. Važne veze označite podatkom ili uslugom koja njima prolazi. Logička komponenta može biti modul unutar jedne poslužiteljske aplikacije: zaseban pravokutnik ne znači zasebno postavljenu uslugu. Tijekom implementacije ažurirajte dijagram kada se promijene odgovornosti ili veze. U završnoj inačici čitatelj treba moći povezati njegove dijelove s kodom. Dijagram razmještaja odgovara na pitanje gdje se ti dijelovi stvarno izvode.

<details>
<summary><strong>Granica sustava, odgovornosti i vanjske ovisnosti</strong></summary>

**Primjer — CrisisMap:** Predložena arhitektura podržava podnošenje prijava kriznih događaja, prikaz na karti i obavijesti o potvrđenim prijavama.

![Dijagram komponenata sustava CrisisMap](./assets/diagrams/4-1-crisismap-system-CMP.svg)

**Slika 4.1. Dijagram komponenata sustava CrisisMap**

**PlantUML izvor:** [izvor](./puml/4-1-crisismap-system-CMP.puml)

| Dio | Odgovornost | Razlog za ovu granicu |
| --- | --- | --- |
| Web-klijent | Prikazuje prijave i prikuplja podatke za novu prijavu. | Korisnička interakcija odvojena je od pravila prijave i trajne pohrane. |
| API za prijave | Prihvaća zahtjeve za rad s prijavama i vraća odgovore. | Odgovornosti izložene putem HTTP-a odvojene su od obrade prijava. |
| Usluga za prijave | Primjenjuje pravila prijave i pristupa pohranjenim prijavama. | Pravila provjere valjanosti i pohrane prijava imaju jasno mjesto odgovornosti. |
| Proces za obavijesti | Određuje korisnike koji trebaju primiti obavijest i traži od vanjskog pružatelja da je pošalje. | Obrada obavijesti odvojena je od odgovaranja na korisnikov zahtjev za prijavu. |

**Analiza primjera:** Vanjska granica odvaja predloženi sustav od pružatelja kartografskih usluga i pružatelja obavijesti. Unutarnja granica grupira API za prijave, uslugu za prijave i proces za obavijesti unutar jedne poslužiteljske aplikacije; njihovi zasebni simboli prikazuju predložene odgovornosti, a ne neovisno postavljene usluge. PostgreSQL pohranjuje podatke aplikacije, dok pružatelj kartografskih usluga osigurava kartografske pločice. Strelice sažimaju ovisnosti, a ne redoslijed transakcije; taj redoslijed prikazuje sekvencijski dijagram. Izostavite proces za obavijesti i Firebase Cloud Messaging ako obavijesti nisu dio odobrenog opsega. Pružatelja prijave u sustav uključite samo ako to zahtijeva odobreni zadatak. Kako implementacija napreduje, ispravite svaku predloženu granicu koja se pokaže netočnom; završni dijagram mora odgovarati onome što je tim izgradio. Jedan koherentan prikaz komponenata korisniji je od nekoliko gotovo jednakih inačica s proturječnim vezama.

</details>

## 4.3 Model podataka i oblikovanje razreda

**Cilj:** Objasniti važne informacije koje aplikacija pohranjuje i jedno reprezentativno područje oblikovanja na razini razreda. Model podataka i UML dijagram razreda odgovaraju na različita pitanja; jedno ne zamjenjuje drugo.

### 4.3.1 ER dijagram i model podataka

Sustav KonfTrack koristi relacijsku PostgreSQL bazu
podataka za pohranu podataka o korisnicima,
konferencijama, znanstvenim radovima, recenzijama
i ostalim povezanim entitetima.

**ER dijagram:**

![ER dijagram sustava KonfTrack](konfTrackERD.png)
**Slika 4.2. ER dijagram sustava KonfTrack**

**Izvor dijagrama:**

[Izvorni ER dijagram](konfTrackERD.pgerd)

**Implementacija:**

Model podataka definiran je u PostgreSQL bazi,
a njegova struktura opisana je u
[Prisma shemi](../backend/prisma/schema.prisma).




### 4.3.2 Reprezentativni UML dijagram razreda

Za reprezentativni dio oblikovanja sustava KonfTrack odabrana je
funkcionalnost predaje i upravljanja znanstvenim radovima.

Dijagram prikazuje osnovne entitete sustava, njihove atribute,
operacije za upravljanje radovima te međusobne veze.

**Dijagram razreda:**

![UML dijagram razreda KonfTrack](./assets/diagrams/4-3-konftrack-paper-CD.svg)

**PlantUML izvor:**

[Izvorni UML dijagram](./puml/4-3-konftrack-paper-CD.puml)

**Referenca na kod pri završnom pregledu:**

[Kontroler za upravljanje radovima](../backend/src/controllers/paperController.js)

[Prisma modeli podataka](../backend/prisma/schema.prisma)

[Servis za provjeru PDF-a](../backend/src/services/pdfService.js)

[Servis za provjeru DOI oznaka](../backend/src/services/crossrefService.js)

**Objašnjenje oblikovanja:**

Razredi Paper, Conference, User, PaperAuthor i PaperVersion
predstavljaju osnovne podatkovne entitete sustava.
Njihove veze prikazane su asocijacijama s odgovarajućim
kardinalnostima.

PaperController odgovoran je za spremanje nacrta,
predaju, dohvat, uređivanje i povlačenje znanstvenih radova.

Prilikom predaje rada koristi PdfService za provjeru
rukopisa i CrossrefService za provjeru DOI oznaka.
Ove ovisnosti prikazane su isprekidanim strelicama.

Kontroler i servisi predstavljaju logičke JavaScript module,
dok podatkovni entiteti odgovaraju modelima Prisma ORM-a.

<details>
<summary><strong>Oblikovanje odgovornosti prijave i dopuštenih promjena</strong></summary>

**Primjer — CrisisMap:** Prijava sadrži pravila za promjenu vlastitog statusa. Razred tijeka rada koordinira podnošenje i pregled, dok sučelje repozitorija opisuje potrebne operacije trajne pohrane. Dijagram predlaže te odgovornosti prije implementacije.

![Dijagram razreda za obradu prijava kriznih događaja](./assets/diagrams/4-3-crisismap-domain-CD.svg)

**Slika 4.3. Dijagram razreda za obradu prijava kriznih događaja**

**PlantUML izvor:** [izvor](./puml/4-3-crisismap-domain-CD.puml)

**Analiza primjera:** `DisasterReport` sadrži dopuštena pravila promjene stanja pa tijek rada ne može prešutno odobriti već odbijenu prijavu. `ReportWorkflow` koordinira operaciju bez preuzimanja pravila prijelaza stanja same prijave; `ReportRepository` definira potrebne operacije pohrane. Kompozicija `Location` i odabrani tipovi prikazuju veze korisne za oblikovanje. Dijagram namjerno izostavlja stupce baze podataka i nepovezane razrede: ER model objašnjava pohranjene podatke, dok ovaj dijagram dodjeljuje programske odgovornosti i operacije. Ti su razredi odluka oblikovanja, a ne obavezni slojevi za svaki tim. Završnu implementaciju usporedite s početnim oblikovanjem i ažurirajte dijagram tamo gdje su se odluke promijenile; generiranje dijagrama iz koda tek na kraju ne može zamijeniti ovo početno modeliranje.

</details>


## 4.4 Ponašanje tijekom izvođenja

**Cilj:** Objasniti jednu važnu interakciju među dijelovima koji se izvode i životni ciklus jednog smislenog objekta. Odaberite primjere koji otkrivaju odluke ili neuspješne putove umjesto izrade dijagrama za svaku CRUD radnju.
### 4.4.1 Reprezentativni UML sekvencijski dijagram

**Cilj:** Prikazati redoslijed poruka za jedan reprezentativni scenarij i odgovore na važnu alternativu.

**Slika 4.4. Sekvencijski dijagram predaje znanstvenog rada**

**Scenarij:** [Naziv interakcije; dodajte UC ID ako postoji.]  
**sekvencijski dijagram:** ![Sekvencijski dijagram](./assets/diagrams/4-4-konftrack-report-submission-SD.svg)  
**PlantUML izvor:** [izvor](./puml/4-4-konftrack-report-submission-SD.puml)

**Opis toka interakcije:**
1. **Slanje obrasca:** Prijavljeni autor putem web-sučelja ispunjava podatke o radu (konferencija, tema, naslov, sažetak, koautori, popis DOI referenci) i prilaže PDF datoteku te šalje zahtjev na `PaperController`.
2. **Provjera ovlasti i roka:** Poslužitelj provjerava aktivnu sesiju/JWT korisnika te je li rok za predaju radova na odabranoj konferenciji još uvijek otvoren. Ako je rok istekao ili podaci nisu potpuni, zahtjev se odbija s porukom o pogrešci.
3. **Provjera PDF rukopisa (`PdfService`):** Kontroler prosljeđuje datoteku servisu `PdfService`, koji provjerava je li datoteka valjani PDF i prelazi li najveći dopušteni broj stranica. Također provjerava sadrže li metapodaci PDF-a ime nekog od autora te u tom slučaju bilježi upozorenje o anonimnosti. Ako datoteka nije valjani PDF ili ima previše stranica, predaja se prekida.
4. **Provjera DOI referenci (`CrossrefService`):** Za svaku unesenu DOI oznaku kontroler poziva `CrossrefService`, koji šalje upit vanjskom Crossref API-ju. Ako DOI ne postoji, sustav bilježi upozorenje. Ako je Crossref API privremeno nedostupan ili istekne vrijeme čekanja (*timeout*), sustav bilježi upozorenje, ali **ne sprječava predaju rada**.
5. **Pohrana datoteke i upis u bazu:** PDF se sprema u mapu `uploads/` pod generiranim nazivom, a putem Prisma ORM-a u PostgreSQL bazu spremaju se zapisi o radu (`Paper`), autorima (`PaperAuthor`) i novoj verziji rukopisa (`PaperVersion`) sa stanjem *predan*.
6. **Odgovor korisniku:** Poslužitelj vraća potvrdu o uspješnoj predaji zajedno s eventualnim upozorenjima (nepostojeći DOI ili ime autora u metapodacima PDF-a).

### 4.4.2 Reprezentativni UML dijagram stanja

**Cilj:** Prikazati važna stanja jednog domenskog objekta ili eksplicitno upravljanog načina rada sustava te događaje ili uvjete koji ih mijenjaju.

**Vlasnik stanja i razlog odabira:** [Navedite objekt ili način rada sustava, kako se njegovo stanje prepoznaje i zašto su prijelazi važni.]  
**Dijagram stanja:** [Ugradite čitljiv UML dijagram stanja.]  
**PlantUML izvor:** [Poveznica na odgovarajuću `.puml` datoteku.]

**Pregled stanja i prijelaza objekta `Paper`:**
* **Nacrt (`draft`):** Početno stanje kada autor spremi rad bez konačne predaje. Obavezna polja i PDF još nisu obavezni, a rad nije vidljiv programskom odboru. Autor može uređivati rad ili ga konačno predati.
* **Predan (`submitted`):** Autor je potvrdio predaju rada prije isteka roka uz valjani PDF rukopis. Rad ulazi u sustav konferencije. Iz ovog stanja autor može povući rad, ili ga predsjednik programskog odbora dodjeljuje recenzentima.
* **U recenziji (`under_review`):** Rad je dodijeljen recenzentima i provodi se dvostruko slijepa recenzija. Autor više ne može mijenjati rukopis, ali i dalje može povući rad.
* **Potrebna revizija (`revision_required`):** Predsjednik programskog odbora na temelju recenzija traži doradu rada. Autor dobiva uvid u anonimizirane recenzije i može predati novu verziju rukopisa (`PaperVersion`), čime se rad vraća u postupak pregleda.
* **Prihvaćen (`accepted`):** Predsjednik programskog odbora donosi odluku o prihvaćanju rada za izlaganje i objavu u programu konferencije.
* **Odbijen (`rejected`):** Konačno stanje kada predsjednik programskog odbora odbije rad na temelju recenzija. Daljnje izmjene niti ponovna predaja istog rada nisu dopuštene.
* **Povučen (`withdrawn`):** Konačno stanje u koje rad prelazi kada ga autor samoinicijativno povuče iz postupka. Rad ostaje evidentiran u bazi radi sljedivosti, ali se isključuje iz daljnje recenzije i programa.
<details>
<summary><strong>Složeno stanje sustava: koordinacija događaja i dostupnost usluge</strong></summary>

**Primjer — CrisisMap:** Ovaj primjer ne prikazuje životni ciklus jednog domenskog objekta, nego dva neovisna aspekta stanja cijelog sustava CrisisMap. Jedna regija opisuje način koordinacije kriznog događaja, a druga opaženu dostupnost usluge. Budući da se ta dva aspekta mogu mijenjati neovisno jedan o drugome, prikazani su kao ortogonalne regije istog složenog stanja sustava.

![Dijagram stanja prijave kriznog događaja](./assets/diagrams/4-5-crisismap-system-SM.svg)

**Slika 4.5. Dijagram stanja sustava s ortogonalnim regijama**

**PlantUML izvor:** [izvor](./puml/4-5-crisismap-system-SM.puml)

**Analiza primjera:** Vlasnik stanja u ovom je primjeru sustav CrisisMap, a ne pojedinačna prijava kriznog događaja. Dvije ortogonalne regije opisuju stanja koja vrijede istodobno: način koordinacije kriznog događaja i dostupnost usluge. Promjena stanja u jednoj regiji ne mora promijeniti stanje u drugoj; primjerice, aktivni krizni događaj može postojati dok je usluga potpuno ili djelomično dostupna.

Prijelazi imaju imenovane događaje ili uvjete, a grane nakon oporavka poslužiteljskog sustava uzimaju u obzir da pružatelj kartografskih usluga može i dalje biti nedostupan. Time primjer pokazuje situaciju u kojoj ortogonalne regije imaju stvarnu semantičku svrhu, a nisu samo način vizualnog grupiranja elemenata.

Ovo je napredniji primjer dijagrama stanja. Koristite takav oblik samo ako sustav stvarno održava dva ili više međusobno neovisnih aspekata stanja koji se mogu mijenjati istodobno. Za projekt u kojem je važan životni ciklus jednog objekta, primjerice prijave, rezervacije ili narudžbe, jednostavniji dijagram stanja tog objekta obično je prikladniji. Složenost dijagrama nije kriterij vrednovanja.

Pri završnom pregledu svaki prikazani prijelaz i uvjet usporedite s ponašanjem koje implementirana aplikacija stvarno podržava.

</details>

## 4.5 Zapisi arhitekturnih odluka

**Cilj:** Sačuvati razloge i obrazložiti odabrane važne arhitekturne odluke kada tablica u odjelku 4.1 nije dovoljna za očuvanje konteksta, razmatranih alternativa i prihvaćenih kompromisa.

Napišite nekoliko ADR-a za važne odluke čije obrazloženje, alternative i kompromis treba sačuvati. Ostale arhitekturne odluke dovoljno je zabilježiti u tablici.

**Detaljni zapis arhitekturne odluke (engl. Architecture Decision Record, ADR), kada odluka zahtijeva dodatno objašnjenje:**

### ADR-[ID]: [Naslov odluke]

| Polje | Zapis odluke |
| --- | --- |
| Kontekst i cilj | [Koji je arhitekturni problem učinio ovu odluku potrebnom?] |
| Kriteriji odluke | [Što je timu stvarno bilo važno?] |
| Razmatrane alternative | [Realne mogućnosti koje je tim razmotrio, ukratko.] |
| Odluka i obrazloženje | [Odabrana mogućnost, zašto je odabrana i čega se tim odrekao.] |
| Status | [Predloženo, prihvaćeno ili zamijenjeno; ako je primjenjivo, poveznica na novu odluku.] |

ADR pišite samo za odluku čije će obrazloženje ostati važno i nakon promjene implementacije. Značajna promjena odluke može se zabilježiti novim ADR-om koji zamjenjuje stari; rutinski odabiri alata ne trebaju detaljne zapise. Promjene dogovorenog opsega pripadaju zahtjevima i njihovoj povijesti promjena; povratna informacija o zadatku ostaje uz odgovarajući zadatak.

<details>
<summary><strong>Odabir jednog poslužiteljskog sustava umjesto zasebno postavljenih usluga</strong></summary>

**Primjer — CrisisMap, ADR-01:**

| ID | Datum | Odluka | Razmatrana alternativa | Obrazloženje i kompromis | Status |
| --- | --- | --- | --- | --- | --- |
| ADR-01 | [Datum odluke] | Jedan poslužiteljski API za obradu prijava | Zasebno postavljene usluge za prijave i resurse | Manji napor postavljanja i integracije za odobreni opseg; kvar poslužiteljskog sustava može utjecati na obje funkcionalnosti. | Prihvaćeno |

### ADR-01: Jedan poslužiteljski API za obradu prijava

| Polje | Zapis odluke |
| --- | --- |
| Kontekst i cilj | Tim tijekom jednog semestra mora izraditi i javno postaviti prijavljivanje kriznih događaja i prikaz usmjeren na kartu. |
| Kriteriji odluke | Tim može izraditi, integrirati, postaviti i objasniti rješenje; komponente i dalje imaju prepoznatljive odgovornosti. |
| Razmatrane alternative | **Jedan poslužiteljski sustav:** moduli dijele aplikaciju koja se postavlja kao cjelina. **Zasebne usluge:** svaka se može izvoditi neovisno, ali zahtijeva dodatni rad na integraciji i postavljanju. |
| Odluka i obrazloženje | Odabrati jedan poslužiteljski sustav s jasno odvojenim odgovornostima vezanim uz prijave. Time se smanjuje rad na postavljanju, uz prihvaćanje da kvar tog sustava može istodobno utjecati na rad s prijavama. |
| Status | Prihvaćeno. |

**Analiza primjera:** Strategija rješenja navodi trenutačni pristup; ADR-01 čuva alternative i kriterije koji su do njega doveli. Dijagrami komponenata i razmještaja trebaju odražavati odabrani pristup kako se sustav razvija. Zaseban ADR koristan je za važnu odluku, a ne za svaki odabir alata.

</details>

## 4.6 Zajednička pravila oblikovanja (engl. Cross-cutting concerns)

**Cilj:** Prepoznati i opisati globalna pravila i mehanizme oblikovanja koji se primjenjujuju u više dijelova aplikacije, navesti gdje se planira primijeniti i ažurirati objašnjenje kako implementacija poprima konačan oblik. Uključite samo pravila koja su važna za ovaj projekt.

**Implementirani mehanizmi:**

| Aspekt | Mehanizam | Pravilo | Mjesto primjene |
| --- | --- | --- | --- |
| [Npr. Sigurnost] | [Npr. Autorizacija korisnika] | [Npr. Ovlasti se provjeravaju na poslužiteljskoj strani za svaki zaštićeni zahtjev; skrivanje elemenata korisničkog sučelja nije dovoljna zaštita.] | [Npr. Middleware za autorizaciju / zaštita API usmjernika.] |
| [Npr. Validacija podataka] | [Npr. Provjera ulaznih podataka] | [Npr. Geoprostorni podaci, uključujući koordinate, provjeravaju se prije pohrane.] | [Npr. Sloj za obradu zahtjeva / odgovarajuća usluga.] |

<details>
<summary><strong>Primjer zajedničkih pravila i mehanizama</strong></summary>

**Primjer — CrisisMap:**

| Aspekt | Mehanizam | Pravilo | Mjesto primjene |
| --- | --- | --- | --- |
| Sigurnost | Autorizacija moderiranja prijava | Ovlasti moderatora provjeravaju se na poslužiteljskoj strani prije odobravanja ili odbijanja prijave; skrivanje gumba u korisničkom sučelju nije dovoljna zaštita. | Poslužiteljski dio funkcionalnosti za moderiranje prijava. |
| Validacija podataka | Provjera lokacije prijave | Nedostajuće ili nevaljane koordinate odbijaju se prije pohrane prijave. | Obrada podnošenja prijave. |
| Aspekt | Mehanizam | Pravilo | Mjesto primjene |
| Upravljanje ovisnostima | Injektiranje ovisnosti | Objekt koji provodi poslovnu logiku ne stvara sam konkretnu implementaciju repozitorija, nego potrebnu ovisnost prima izvana. | `ReportWorkflow` prima implementaciju sučelja `ReportRepository` pri stvaranju objekta. |

**Analiza primjera:** Prvi redak pokazuje sigurnosno pravilo koje se provodi neovisno o korisničkom sučelju, a drugi pravilo provjere podataka prije pohrane. U vlastitoj dokumentaciji navedite samo aspekte koji se stvarno primjenjuju u projektu te konkretno mjesto njihove primjene.

Obrasci poput injektiranja ovisnosti (engl. Dependency Injection) ili repozitorija (engl. Repository) pripadaju ovdje samo kada ih kod stvarno koristi i kada objašnjavaju zajedničko pravilo oblikovanja. Tvrdnja „sustav koristi autentikaciju” sama po sebi ne govori čitatelju gdje se provjera provodi niti što štiti. Sigurnost, bilježenje događaja i obrada pogrešaka ne trebaju svaki zaseban pododjeljak po zadanom pravilu.

</details>

## 4.7 Arhitekturni rizici i tehnički dug

**Cilj:** Navesti stvarne tehničke ili domenske rizike i svjesno prihvaćene kompromise oblikovanja, njihove posljedice i način na koji ih tim obrađuje.

**Tehnički rizici i dug**

| Rizik ili kompromis | Posljedica | Odgovor na rizik  |
| --- | --- | --- |
| *Npr. Pohrana slika u lokalni file sustav* | *Pri ponovnom pokretanju kontejnera slike će biti izgubljene.* | *Prihvaćeni tehnički dug za demo verziju. U budućnosti planirana migracija na S3 (AWS).* |
| *Npr. Nepostojanje paginacije na API-ju* | *Moguće usporenje klijenta pri dohvaćanju velike količine podataka iz baze.* | *Ograničen broj testnih unosa. Dodavanje paginacije ostavljeno za sljedeću iteraciju.* |
|  |  |  |

**Poslovni ili domenski rizici, ako su značajni**

| Rizik | Posljedica | Odgovor na rizik |
| --- | --- | --- |
| **Jedna instanca aplikacije obrađuje prijave** | Kvar poslužitelja (hostinga) privremeno onemogućuje nove prijave. | Prihvaćeno u okviru ograničenja projekta za kolegij; tim ne tvrdi da postoji automatsko prebacivanje na pričuvni sustav (failover). |
| **Nedostupnost vanjskke kartografske usluge** | Prikaz karte može prestati raditi iako su pohranjene prijave u bazi i dalje dostupne. | Implementirana je korisniku vidljiva poruka o pogrešci uz alternativu pregleda prijava u tekstualnom obliku. |
|  |  |  |

<details>
<summary><strong>Primjer tehničkih i domenskih rizika</strong></summary>

**Primjer — CrisisMap, tehnički rizici:**

| Rizik ili kompromis | Posljedica | Odgovor na rizik |
| --- | --- | --- |
| Jedna instanca aplikacije obrađuje podnošenje prijava. | Kvar smještaja aplikacije privremeno onemogućuje nove prijave. | Rizik je prihvaćen za opseg projekta; ne uvodi se pričuvna instanca ni automatsko prebacivanje. |
| Vanjske kartografske usluge nisu dostupne. | Prikaz karte može prestati raditi iako su pohranjene prijave i dalje dostupne. | Karta se ne može prikazati, ali pohranjene prijave ostaju dostupne. | Aplikacija prikazuje poruku o nedostupnosti karte i omogućuje pregled prijava u tekstualnom obliku. |

**Primjer — CrisisMap, domenski rizik:** Čitatelji mogu neprovjerenu prijavu građanina zamijeniti za službeno upozorenje. Tim može u sučelju jasno razlikovati status provjere prijave i uputiti korisnike na službene izvore.

**Analiza primjera:** Prvi tehnički rizik prihvaća realistično ograničenje projekta kolegija; domenski se rizik odnosi na način na koji čitatelj tumači prijavu. Zabilježite ograničenje i realan odgovor koji pripadaju vašem projektu. Neostvarena obavezna funkcionalnost nije tehnički dug nego nedovršen opseg.

</details>

---


>**Završna provjera:** Usporedite dijagrame komponenata, podataka, razreda, sekvencijski dijagram i dijagram stanja s implementiranim kodom, ponašanjem aplikacije i stvarnim razmještajem. Ažurirajte zastarjele opise oblikovanja i objasnite svaku važnu razliku koja ostaje; poveznice na odgovarajući kod ili ispitivanja korisne su kada odgovaraju na konkretno pitanje.
