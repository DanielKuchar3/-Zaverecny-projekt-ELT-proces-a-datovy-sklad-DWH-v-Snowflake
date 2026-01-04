# Zaverecny projekt ELT proces a datovy sklad DWH v Snowflake

 ## **Úvod a popis zdrojových dát** 

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
  <img src="./dimenzionalny model.png" alt="Hviezdicová schéma">
  <br>
  <em>Obrázok 2 Dimenzionálny model</em>
</p>
