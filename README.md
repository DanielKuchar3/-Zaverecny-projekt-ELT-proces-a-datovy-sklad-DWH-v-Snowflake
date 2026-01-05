# Projekt - OAG_GLOBAL_AIRLINE_SCHEDULES


 ## **1. Úvod a popis zdrojových dát** 

Vybrali sme si dataset od spoločnosti OAG (dostupný cez Snowflake Marketplace), pretože poskytuje komplexný pohľad na logistiku leteckej dopravy vrátane detailných informácií o konfigurácii sedadiel a trasách.

Dáta podporujú proces Plánovania leteckých kapacít a optimalizácie sietí (Network Planning). Tento proces umožňuje aerolinkám a letiskám analyzovať ponuku miest na trhu a efektivitu využitia lietadiel.

**Dataset obsahuje štruktúrované údaje, ktoré môžeme rozdeliť do niekoľkých kategórií:**

**Identifikačné údaje letov:** Čísla letov, ICAO kódy dopravcov a unikátne identifikátory (fingerprints).

**Časové údaje:** Dátumy letov, časy odletov a príletov, celkový uplynutý čas (Elapsed Time).

**Kapacitné údaje:** Celkový počet sedadiel rozdelený podľa cestovných tried (Economy, Business, First Class, Premium Economy).

**Geografické údaje:** Mestá a krajiny odletu/príletu, ICAO kódy letísk a celková vzdialenosť (Distance).




Cieľom analýzy je transformovať surové dáta do formy hviezdicovej schémy (Star Schema). Analýza sa zameriava na:
Porovnanie celkového počtu prepravených ľudí u hlavných aerolínií.
Sledovanie priemerných dĺžok letov a geografického rozloženia dopravy.
Identifikácia vyťaženosti leteckej siete v rámci jednotlivých dní v týždni.
Analýza najvýznamnejších svetových letísk z hľadiska hustoty prevádzky.

V povodnej tabulke sa nachadzaju udaje o priletoch a odletoch,
údaje o konrétnych typoch lietadla ako aj názov leteckej spoločnosti.
Taktiež sa tu nachádzajú údaje o rôznych triedach pre cestujúcich (economy,first class,businnes class) a dátumoch letov.


<p align="center">
  <img src="./erd diagram.png" alt="ERD Schema">
  <br>
  <em>Obrázok 1 erd diagram</em>
</p>

---

## **2. Návrh dimenzionálneho modelu**

Model je navrhnutý ako klasická hviezdicová schéma, kde centrálna tabuľka faktov je prepojená s viacerými dimenziami pomocou vzťahu 1:N.

**Tabuľka faktov:** fact_flights

Tabuľka faktov uchováva kvantitatívne údaje o jednotlivých letoch a slúži ako centrálny bod analýzy.

**Primárny kľúč:** oag_schedule_fingerprint (unikátny identifikátor letového poriadku).

**Cudzie kľúče:**
dim_date_id (prepojenie na časovú os).
dim_carrier_id (prepojenie na leteckého dopravcu).
dim_aircraft_id (prepojenie na typ lietadla).
dim_departure_id a dim_arrival_id (prepojenie na geografické údaje).
dim_service_id, dim_tons_id.

**Tabuľky dimenzíí:**
Pre každú dimenziu sme zvolili SCD Typ 0 (Original), pretože letecké poriadky v tomto datasete sú historické záznamy, kde sa atribúty spätne nemenia. V prípade potreby sledovania zmien (napr. zmena názvu aerolinky v čase) by sa využil SCD Typ 2.

**`dim_date`** - časové atribúty(dátum letu, deň v týždni, časový posun)

**`dim_carrier`** - Informácie o dopravcovi (názov, ICAO kód, vlastník lietadla, číslo letu).

**`dim_aircraft`** - Technické parametre lietadla (typ, výrobca, názov modelu meno spoločnosti).

**`dim_departure`** - Detail letiska odletu (mesto, krajina, kód portu, čas odletu).

**`dim_arrival`** - Detail letiska príletu (mesto, krajina, kód portu, čas príletu).

**`dim_service`** - Informácie o type servisu (palubné jedlo, trieda nákladu)




<p align="center">
  <img src="./erd_diagramy/projekt_star_erd_diagram.png" alt="Hviezdicová schéma">
  <br>
  <em>Obrázok 2 Dimenzionálny model</em>
</p>

---

## **3. ELT proces v Snowflake**
# **Extract**


Dáta boli získané zo Snowflake Marketplace. Bol vybraný dataset globálnych leteckých poriadkov od spoločnosti OAG.

Zdrojová databáza: **`OAG_GLOBAL_AIRLINE_SCHEDULES_SAMPLE`**

Zdrojová schéma: **`PUBLIC`**

Zdrojová tabuľka: **`OAG_SCHEDULE`**

Prvým krokom bolo vytvorenie staging tabuľky, ktorá slúži ako dočasné úložisko pre surové dáta pred ich ďalším spracovaním.
```sql
CREATE OR REPLACE TABLE flights AS
SELECT * FROM oag_global_airline_schedules_sample.public.oag_schedule;
```
 Tento príkaz skopíruje všetky záznamy zo zdieľaného datasetu do našej lokálnej tabuľky flights, čím umožní vykonávať transformácie bez ovplyvnenia zdrojových dát.



# **Transform**


V tejto fáze dochádza k čisteniu dát, deduplikácii a tvorbe dimenzií. Využili sme techniku SCD Typ 0, kde sú historické údaje zachované v pôvodnom stave.

Tvorba dimenzií a čistenie
Pri tvorbe dimenzií sme použili klauzulu GROUP BY na odstránenie duplicít a funkciu DENSE_RANK() na generovanie unikátnych kľúčov.

**Príklad transformácie (Dimenzia dopravcov):**
```sql
CREATE OR REPLACE TABLE dim_carrier AS
SELECT
    DENSE_RANK() OVER (ORDER BY carrier_cd_icao) AS id,
    carrier,
    carrier_cd_icao,
    operating,
    acft_owner,
    fltno
FROM flights
GROUP BY carrier, carrier_cd_icao, operating, acft_owner, fltno
ORDER BY carrier_cd_icao;
```

 DENSE_RANK() zabezpečí pridelenie unikátneho sekvenčného ID každému unikátnemu dopravcovi. GROUP BY zabezpečí, že každá kombinácia atribútov sa v dimenzii nachádza iba raz.

**Dimenzia časov:**
```sql
 CREATE OR REPLACE TABLE dim_date AS
SELECT
    DENSE_RANK() OVER (ORDER BY flight_date) AS id,
    flight_date,
    arrday,
    file_date,
    elptim
FROM flights
GROUP BY flight_date, arrday, file_date, elptim
ORDER BY flight_date;
```

**Dimenzia lietadiel:**
```sql
CREATE OR REPLACE TABLE dim_aircraft AS
SELECT
    DENSE_RANK() OVER (ORDER BY equipment_cd_icao) AS id,
    equipment_cd_icao,
    genacft,
    inpacft,
    sad_name,
    sad,
    dupcarfl
FROM flights
GROUP BY equipment_cd_icao, genacft, inpacft, sad_name, sad, dupcarfl
ORDER BY equipment_cd_icao;
```

**Dimenzia služieb:**
```sql
CREATE OR REPLACE TABLE dim_service AS
SELECT
    DENSE_RANK() OVER (ORDER BY service) AS id,
    service,
    px,
    meals,
    frtclass
FROM flights
GROUP BY service, px, meals, frtclass
ORDER BY service;
```

**Dimenzia odletov:**
```sql
CREATE OR REPLACE TABLE dim_departure AS
SELECT
    DENSE_RANK() OVER (ORDER BY dep_port_cd_icao) AS id,
    depcity,
    depctry,
    depapt,
    dep_port_cd_icao,
    deptim
FROM flights
GROUP BY depcity, depctry, depapt, dep_port_cd_icao, deptim
ORDER BY dep_port_cd_icao;

```

**Dimenzia príletov:**
```sql
CREATE OR REPLACE TABLE dim_arrival AS
SELECT
    DENSE_RANK() OVER (ORDER BY arr_port_cd_icao) AS id,
    arrcity,
    arrctry,
    arrapt,
    arr_port_cd_icao,
    arrtim
FROM flights
GROUP BY arrcity, arrctry, arrapt, arr_port_cd_icao, arrtim
ORDER BY arr_port_cd_icao;

```

**Dimenzia hmotnosti:**
```sql
CREATE OR REPLACE TABLE dim_tons AS
SELECT
    DENSE_RANK() OVER (ORDER BY tons) AS id,
    tons
FROM flights
GROUP BY tons
ORDER BY tons;

```


# **Load**


V záverečnej fáze ELT procesu sme naplnili faktovú tabuľku fact_flights. Tento krok transformuje ploché dáta zo stagingu do relačnej štruktúry.

**Hlavný sql príkaz:**
```sql
CREATE OR REPLACE TABLE fact_flights AS
SELECT
    f.oag_schedule_fingerprint,
    f.total_seats,
    f.distance,
    f.economy_class_seats,
    ROUND((f.economy_class_seats * 100.0 / NULLIF(f.total_seats, 0)), 2) AS economy_class_pct,
    f.economy_plus_class_seats,
    f.premium_economy_class_seats,
    f.business_class_seats,
    f.first_class_seats,
    d.id AS dim_date_id,
    s.id AS dim_service_id,
    a.id AS dim_aircraft_id,
    c.id AS dim_carrier_id,
    dep.id AS dim_departure_id,
    arr.id AS dim_arrival_id,
    t.id AS dim_tons_id
FROM flights f

-- DATE
JOIN dim_date d
  ON f.flight_date = d.flight_date
 AND f.arrday = d.arrday
 AND f.file_date = d.file_date
 AND f.elptim = d.elptim

-- SERVICE
JOIN dim_service s
  ON f.service = s.service
 AND f.px = s.px
 AND f.meals = s.meals
 AND f.frtclass = s.frtclass

-- AIRCRAFT
JOIN dim_aircraft a
  ON f.equipment_cd_icao = a.equipment_cd_icao
 AND f.genacft = a.genacft
 AND f.inpacft = a.inpacft
 AND f.sad_name = a.sad_name
 AND f.sad = a.sad
 AND f.dupcarfl = a.dupcarfl

-- CARRIER
JOIN dim_carrier c
  ON f.carrier_cd_icao = c.carrier_cd_icao
 AND f.carrier = c.carrier
 AND f.operating = c.operating
 AND f.acft_owner = c.acft_owner
 AND f.fltno = c.fltno

-- DEPARTURE
JOIN dim_departure dep
  ON f.dep_port_cd_icao = dep.dep_port_cd_icao
 AND f.depcity = dep.depcity
 AND f.depctry = dep.depctry
 AND f.depapt = dep.depapt
 AND f.deptim = dep.deptim

-- ARRIVAL
JOIN dim_arrival arr
  ON f.arr_port_cd_icao = arr.arr_port_cd_icao
 AND f.arrcity = arr.arrcity
 AND f.arrctry = arr.arrctry
 AND f.arrapt = arr.arrapt
 AND f.arrtim = arr.arrtim

-- TONS
JOIN dim_tons t
  ON f.tons = t.tons;
```
---

## **4. Vizualizácia dát**

Dashboard obsahuje 5 kľúčových vizualizácií, ktoré poskytujú prehľad o kapacitách, operačnej efektivite a geografickom rozložení letov. Tieto vizualizácie umožňujú manažmentu leteckých spoločností a letísk lepšie pochopiť dynamiku trhu.

<p align="center">
  <img src="./grafy.png" alt="grafy">
  <br>
  <em>Obrázok 3 Dashboard grafov</em>
</p>



**Graf 1: Top 10 najdlhších leteckých trás (vzdialenosť):**

<p align="center">
  <img src="./10 najdlhsich tras.png" alt="grafy">
  <br>
  <em>Obrázok 4 graf 1</em>
</p>

Táto vizualizácia identifikuje najdlhšie trasy (long-haul), najdlhšia bola z Wuhanu(Čína) do Santiago (Ćile)

```sql
SELECT 
    dep.depapt || ' -> ' || arr.arrapt AS route,
    MAX(f.distance) AS route_distance
FROM fact_flights f
JOIN dim_departure dep ON f.dim_departure_id = dep.id
JOIN dim_arrival arr ON f.dim_arrival_id = arr.id
GROUP BY route
ORDER BY route_distance DESC
LIMIT 10;

```


**Graf 2: Top 10 leteckých spoločností podľa počtu prepravených ľudí:**

<p align="center">
  <img src="./10 leteckych spolocnosti.png" alt="grafy">
  <br>
  <em>Obrázok 5 graf 2</em>
</p>

Táto vizualizácia odpovedá na otázku, ktoré letecké spoločnosti dominujú trhu z hľadiska objemu prepravnej kapacity. Najviac ľudí prepravilo American Airlines.Druhý za ním je Delta Airlines.

```sql
SELECT 
    a.sad_name AS airline_name, 
    SUM(COALESCE(f.total_seats, 0)) AS total_capacity
FROM fact_flights f
JOIN dim_aircraft a ON f.dim_aircraft_id = a.id
GROUP BY airline_name
HAVING total_capacity > 0
ORDER BY total_capacity DESC
LIMIT 10;
```

**Graf 3:Počet odletov podľa krajín:**

<p align="center">
  <img src="./pocet odletov podla krajin.png" alt="grafy">
  <br>
  <em>Obrázok 6 graf 3</em>
</p>

Táto vizualizácia poskytuje prehľad o tom, ktoré krajiny generujú najväčší objem leteckej dopravy. Identifikuje kľúčové trhy a geografické centrá, z ktorých lietadlá najčastejšie štartujú. Najviac odletelo z Veľkej Británie

```sql
SELECT 
    dep.depctry AS departure_country,
    COUNT(*) AS number_of_flights
FROM fact_flights f
JOIN dim_departure dep ON f.dim_departure_id = dep.id
GROUP BY departure_country
ORDER BY number_of_flights DESC
LIMIT 10;
```

**Graf 4:Priemerná dlžka letu z danej krajiny:**

<p align="center">
  <img src="./avg dlzka letu.png" alt="grafy">
  <br>
  <em>Obrázok 7 graf 4</em>
</p>

Táto vizualizácia analyzuje efektivitu a charakter trás v jednotlivých krajinách. Pomáha identifikovať krajiny, ktoré sú primárne orientované na diaľkovú prepravu (long-haul) v porovnaní s krajinami s prevahou regionálnych letov.

```sql
SELECT 
    depctry AS country, 
    ROUND(AVG(NULLIF(elptim, 0)), 2) AS avg_duration
FROM flights
WHERE depctry IS NOT NULL 
  AND elptim > 0
GROUP BY country
ORDER BY avg_duration DESC
LIMIT 15;
```

**Graf 5:Počet letov podľa dňa v týždni:**

<p align="center">
  <img src="./pocet letov podla dna.png" alt="grafy">
  <br>
  <em>Obrázok 8 graf 5</em>
</p>

Táto vizualizácia sleduje počet letov v danom dni. Identifikuje dni s najvyšším náporom na kapacitu, čo je kľúčové pre operačné plánovanie letísk a optimalizáciu letových poriadkov leteckých spoločností.


```sql
SELECT 
    d.arrday AS day_number,
    -- Prevod čísla dňa na zrozumiteľný názov pre graf
    CASE 
        WHEN d.arrday = '1' THEN 'Pondelok'
        WHEN d.arrday = '2' THEN 'Utorok'
        WHEN d.arrday = '3' THEN 'Streda'
        WHEN d.arrday = '4' THEN 'Štvrtok'
        WHEN d.arrday = '5' THEN 'Piatok'
        WHEN d.arrday = '6' THEN 'Sobota'
        WHEN d.arrday = '7' THEN 'Nedeľa'
    END AS day_name,
    -- SUM spočíta všetky sedadlá, TRY_TO_NUMBER ošetrí chybu s textom ''
    SUM(COALESCE(TRY_TO_NUMBER(f.total_seats), 0)) AS total_capacity
FROM fact_flights f
JOIN dim_date d ON f.dim_date_id = d.id
GROUP BY day_number, day_name
ORDER BY day_number;
```
---

**Autor:** Daniel Kuchar   ,   Juraj Švajda

---



