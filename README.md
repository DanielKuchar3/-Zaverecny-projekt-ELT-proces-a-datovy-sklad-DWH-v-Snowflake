# -Zaverecny-projekt-ELT-proces-a-datovy-sklad-DWH-v-Snowflake

Úvod a popis zdrojových dát

Vybrali sme si dataset od spoločnosti OAG (dostupný cez Snowflake Marketplace), pretože poskytuje komplexný pohľad na logistiku leteckej dopravy vrátane detailných informácií o konfigurácii sedadiel a trasách.

Dáta podporujú proces Plánovania leteckých kapacít a optimalizácie sietí (Network Planning). Tento proces umožňuje aerolinkám a letiskám analyzovať ponuku miest na trhu a efektivitu využitia lietadiel.

Dataset obsahuje štruktúrované údaje, ktoré môžeme rozdeliť do niekoľkých kategórií:

Identifikačné údaje letov: Čísla letov, ICAO kódy dopravcov a unikátne identifikátory (fingerprints).
Časové údaje: Dátumy letov, časy odletov a príletov, celkový uplynutý čas (Elapsed Time).
Kapacitné údaje: Celkový počet sedadiel rozdelený podľa cestovných tried (Economy, Business, First Class, Premium Economy).
Geografické údaje: Mestá a krajiny odletu/príletu, ICAO kódy letísk a celková vzdialenosť (Distance).

Cieľom analýzy je transformovať surové dáta do formy hviezdicovej schémy (Star Schema), ktorá umožní rýchle a efektívne reportovanie. Analýza sa zameriava na:
Porovnanie celkového počtu prepravených ľudí u hlavných aerolínií.
Sledovanie priemerných dĺžok letov a geografického rozloženia dopravy.
Identifikácia vyťaženosti leteckej siete v rámci jednotlivých dní v týždni.
Analýza najvýznamnejších svetových letísk z hľadiska hustoty prevádzky.



<p align="center">
  <img src="./erd diagram.png" alt="ERD Schema">
  <br>
  <em>Obrázok 1 erd diagram</em>
</p>

