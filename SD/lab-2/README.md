# Laboratorul 2 — Cuantificarea citirilor Dashboard-ului

Personal Investment Dashboard permite unui Utilizator autentificat să urmărească informații despre piață și investițiile sale. Sistemul extern principal este **Market Data Provider**, care furnizează datele financiare necesare.

Pentru acest laborator sunt analizate următoarele citiri:

- Overview;
- Filter;
- Stock price;
- History;
- Watchlist;
- Search.

---

# 1. Cerințe de calitate

<!-- O cerință de calitate este formulată folosind: -->

```text
măsură + țintă + condiție de operare
```

Latența, disponibilitatea, debitul și consistența trebuie definite prin rezultate măsurabile, nu prin afirmații generale precum „sistemul trebuie să fie rapid”.

### 1.1. Latența citirilor

Latența este măsurată din momentul în care Dashboard-ul primește cererea de citire până în momentul în care trimite rezultatul corect sau un rezultat explicit indisponibil.

**Cerință:**

> În perioadele aglomerate, pentru citirile valide ale Dashboard-ului, **p95 al latenței trebuie să fie ≤ 2 secunde**.

Aceasta înseamnă că cel puțin 95% dintre citiri trebuie finalizate în maximum două secunde.

Pentru Stock price, unde utilizatorul se așteaptă la un rezultat care să apară imediat, folosim suplimentar:

```text
p50 <= 500 ms
p95 <= 2 s
```

Dacă un rezultat corect nu poate fi furnizat în limita de 2 secunde, citirea nu respectă ținta de latență. Dacă informația necesară nu este disponibilă, Dashboard-ul trebuie să returneze un rezultat explicit **indisponibil**, nu să inventeze o valoare.

Un răspuns rapid „indisponibil” poate respecta cerința de latență, dar nu înseamnă automat că celelalte cerințe de calitate sunt respectate.

### 1.2. Disponibilitate

Dashboard-ul este considerat **utilizabil** atunci când un Utilizator autentificat poate efectua citirile valide și poate primi un rezultat corect sau un rezultat alternativ permis de specificație.

Pentru prima versiune, orele de tranzacționare sunt definite ca **09:30–16:00 ET în zilele de tranzacționare**, corespunzător sesiunii principale a pieței americane.

Alegem două ținte:

| Interval              |      Țintă |
| --------------------- | ---------: |
| Ore de tranzacționare | **99.99%** |
| Restul zilei          |  **99.9%** |

Orele de tranzacționare primesc o țintă mai mare deoarece atunci datele de piață se modifică activ și este de așteptat ca Dashboard-ul să fie folosit mai intens. În afara acestor ore, o țintă de 99.9% este suficientă pentru prima versiune.

Pentru calcul folosim ca ipoteză un interval de 30 zile cu aproximativ **22 zile de tranzacționare**.

```text
Ore de tranzacționare:
22 zile × 6.5 ore = 143 ore

Buget indisponibilitate:
143 h × (1 - 0.9999)
= 0.0143 h
≈ 51.5 secunde
```

Pentru restul zilei:

```text
30 × 24 = 720 ore

720 - 143 = 577 ore

577 h × (1 - 0.999)
= 0.577 h
≈ 34 minute 37 secunde
```

Rezultă:

| Interval măsurat      | Timp măsurat | Disponibilitate | Buget de indisponibilitate |
| --------------------- | -----------: | --------------: | -------------------------: |
| Ore de tranzacționare |        143 h |          99.99% |             ≈ 51.5 secunde |
| Restul zilei          |        577 h |           99.9% |              ≈ 34 min 37 s |

Numărul real de zile de tranzacționare dintr-un interval de 30 zile poate varia; 22 este o ipoteză pentru estimarea laboratorului.

### 1.2.1 Disponibilitate (Opțiune cu cerințe mai ușor realizabile)
 
| Interval              | Țintă   |
| --------------------- | ------- |
| Ore de tranzacționare | **99%** |
| Restul timpului       | **98%** |

**Ore de tranzacționare:**

```
Timp măsurat:
22 zile × 6,5 ore = 143 ore

Disponibilitate țintă:
99% = 0,99

Buget de indisponibilitate:
143 h × (1 − 0,99)
= 143 h × 0,01
= 1,43 h
= 1 oră 25 minute 48 secunde
```

**Restul timpului:**

Intervalul include orele din afara sesiunilor de tranzacționare și zilele fără tranzacționare.

```
Timp total:
30 zile × 24 ore = 720 ore

Timp în afara sesiunilor de tranzacționare:
720 − 143 = 577 ore

Disponibilitate țintă:
98% = 0,98

Buget de indisponibilitate:
577 h × (1 − 0,98)
= 577 h × 0,02
= 11,54 h
= 11 ore 32 minute 24 secunde
```

Rezultă:

|Interval măsurat|Timp măsurat|Disponibilitate|Buget de indisponibilitate|
|---|---|---|---|
|Ore de tranzacționare|143 h|99%|1 h 25 min 48 s|
|Restul timpului|577 h|98%|11 h 32 min 24 s|

### 1.3. Consistența Stock price

Condițiile clientului precizează că întârzierea așteptată a furnizorului este de aproximativ **15 minute** și că datele mai vechi trebuie marcate ca întârziate sau indisponibile.

Stabilim următoarea regulă:

> Pentru Stock price, Dashboard-ul poate afișa ultimul preț acceptat dacă vechimea acestuia este de maximum **15 minute**, împreună cu ora furnizorului. Dacă datele depășesc 15 minute, ele sunt marcate ca întârziate. Dacă nu există un preț valid disponibil, rezultatul este afișat ca **indisponibil**, niciodată ca `0`.

Astfel, utilizatorul poate diferenția între:

```text
preț actual acceptat
preț întârziat
preț indisponibil
```

Această regulă urmează principiul conform căruia datele vechi pot fi permise numai într-o limită definită și trebuie etichetate corespunzător.

### 1.4. Consistența Watchlist

Pentru Watchlist folosim regula **read-your-writes**.

> După ce Dashboard-ul confirmă adăugarea sau eliminarea unui Stock din Watchlist, următoarea citire a Watchlist-ului făcută de același Utilizator trebuie să reflecte modificarea confirmată.

De asemenea:

> Un Utilizator autentificat nu poate citi sau modifica Watchlist-ul altui Utilizator.

### 1.5. Debit

Debitul reprezintă numărul de citiri acceptabile finalizate pe secundă, în timp ce celelalte cerințe de calitate continuă să fie respectate.

Ținta de debit va fi determinată de estimarea traficului la deschiderea pieței:

| Utilizatori concurenți | Debit minim țintă |
| ---------------------: | ----------------: |
|                    300 |       **103 RPS** |
|                  3.000 |     **1.030 RPS** |
|                 30.000 |    **10.296 RPS** |

Aceste valori sunt calculate în secțiunile următoare.

---

# 2. Estimări RPS în regim stabil

Folosim formula cerută:

```text
RPS = Utilizatori concurenți × participare × acțiuni per Utilizator / secunde
```

Comportamentul Utilizatorului este cel oferit în condițiile laboratorului.

## 2.1. Overview

70% dintre Utilizatori reîmprospătează o dată la 30 secunde.

```text
300 × 0.70 / 30 = 7 RPS
3,000 × 0.70 / 30 = 70 RPS
30,000 × 0.70 / 30 = 700 RPS
```

## 2.2. Filter

50% modifică filtrul de 3 ori pe minut.

```text
300 × 0.50 × 3 / 60 = 7.5 RPS
3,000 × 0.50 × 3 / 60 = 75 RPS
30,000 × 0.50 × 3 / 60 = 750 RPS
```

## 2.3. Stock price

20% solicită Stock price o dată pe secundă.

```text
300 × 0.20 / 1 = 60 RPS
3,000 × 0.20 / 1 = 600 RPS
30,000 × 0.20 / 1 = 6,000 RPS
```

## 2.4. History

20% solicită History o dată la 5 minute.

```text
5 minute = 300 secunde

300 × 0.20 / 300 = 0.2 RPS
3,000 × 0.20 / 300 = 2 RPS
30,000 × 0.20 / 300 = 20 RPS
```

## 2.5. Watchlist

60% reîmprospătează o dată pe minut.

```text
300 × 0.60 / 60 = 3 RPS
3,000 × 0.60 / 60 = 30 RPS
30,000 × 0.60 / 60 = 300 RPS
```

## 2.6. Search

10% caută de 3 ori pe minut.

```text
300 × 0.10 × 3 / 60 = 1.5 RPS
3,000 × 0.10 × 3 / 60 = 15 RPS
30,000 × 0.10 × 3 / 60 = 150 RPS
```

### Rezultatul complet

| Citire                    | 300 Utilizatori | 3.000 Utilizatori | 30.000 Utilizatori |
| ------------------------- | --------------: | ----------------: | -----------------: |
| Overview                  |           7 RPS |            70 RPS |            700 RPS |
| Filter                    |         7.5 RPS |            75 RPS |            750 RPS |
| Stock price               |          60 RPS |           600 RPS |          6.000 RPS |
| History                   |         0.2 RPS |             2 RPS |             20 RPS |
| Watchlist                 |           3 RPS |            30 RPS |            300 RPS |
| Search                    |         1.5 RPS |            15 RPS |            150 RPS |
| **Total în regim stabil** |    **79.2 RPS** |       **792 RPS** |      **7.920 RPS** |

---

# 3. Estimări RPS la deschiderea pieței

La deschiderea pieței:

- 30% dintre Utilizatori reîmprospătează Overview o dată într-un interval de 10 secunde;
- 60% din acest grup reîmprospătează și Watchlist;
- aceste acțiuni sunt adăugate peste traficul stabil;
- după subtotal se adaugă o marjă de capacitate de 10%;
- numai ținta finală este rotunjită în sus.

### Flux suplimentar Overview

```text
300 × 0.30 / 10 = 9 RPS
3,000 × 0.30 / 10 = 90 RPS
30,000 × 0.30 / 10 = 900 RPS
```

### Flux suplimentar Watchlist

```text
300 × 0.30 × 0.60 / 10 = 5.4 RPS
3,000 × 0.30 × 0.60 / 10 = 54 RPS
30,000 × 0.30 × 0.60 / 10 = 540 RPS
```

### Calcul complet

| Calcul la deschiderea pieței       | 300 Utilizatori | 3.000 Utilizatori | 30.000 Utilizatori |
| ---------------------------------- | --------------: | ----------------: | -----------------: |
| Trafic stabil de citire            |            79.2 |               792 |              7.920 |
| Flux suplimentar Overview          |               9 |                90 |                900 |
| Flux suplimentar Watchlist         |             5.4 |                54 |                540 |
| **Subtotal la deschiderea pieței** |        **93.6** |           **936** |          **9.360** |
| Marjă de capacitate 10%            |            9.36 |              93.6 |                936 |
| Subtotal × 1.10                    |          102.96 |           1.029.6 |             10.296 |
| **Țintă rotunjită în sus**         |     **103 RPS** |     **1.030 RPS** |     **10.296 RPS** |

Exemplu pentru nivelul de 3.000 Utilizatori:

```text
792 + 90 + 54 = 936 RPS

936 × 1.10 = 1,029.6 RPS

Ținta finală = 1,030 RPS
```

`Stock price` reprezintă cea mai mare parte a traficului stabil:

```text
600 / 792 × 100 ≈ 75.8%
```

Prin urmare, Stock price este primul flux care merită investigat pentru presiune asupra debitului și latenței. 

---

# 4. Estimări de stocare

## 4.1. Definirea unui Stock

Investor.gov definește stock-ul ca un tip de security care oferă deținătorului o parte din proprietatea unei companii și identifică două categorii principale: **common stock** și **preferred stock**.

Pentru prima versiune a Personal Investment Dashboard definim un **Stock** astfel:

> Un Stock este o acțiune comună (`common stock`) a unei companii listate activ pe Nasdaq Stock Market sau New York Stock Exchange, identificată prin simbolul său de tranzacționare.

### Domeniul primei versiuni

| Decizie                        | Alegere                                                                   |
| ------------------------------ | ------------------------------------------------------------------------- |
| Țară                           | SUA                                                                       |
| Piețe principale               | Nasdaq și NYSE                                                            |
| Common stock                   | Inclus                                                                    |
| Preferred stock                | Exclus                                                                    |
| ADR                            | Exclus din prima versiune                                                 |
| ETF                            | Exclus                                                                    |
| Funds                          | Exclus                                                                    |
| Instrumente inactive/delistate | Păstrate numai în istoricul existent; nu pot fi adăugate ca Stocks active |
| Monedă principală              | USD                                                                       |

Limitarea la common stocks simplifică prima versiune și este compatibilă cu obiectivul primului laborator de monitorizare a investițiilor fără extinderea Dashboard-ului într-un produs complet de tranzacționare.

## 4.2. Numărul estimat de Stocks

La **27 septembrie 2026**, Nasdaq declară că Nasdaq Stock Market găzduiește peste **3.300 de corporate listings**.

NYSE declară că are o comunitate de peste **2.400 de emitenți**.

Pentru estimarea de capacitate folosim:

```text
3,300 + 2,400 ≈ 5,700 Stocks
```

Prin urmare:

> **Ipoteză de proiectare: prima versiune trebuie să poată păstra aproximativ 5.700 Stocks.**

Această valoare este o **estimare de planificare**, nu numărul exact de common stocks: statisticile publice ale burselor sunt exprimate în listings/emitenți și pot include categorii care nu intră în definiția noastră. Într-un sistem real, lista exactă ar trebui derivată din directorul de instrumente al furnizorului selectat.

---

## 4.3. Date sincronizate

Stocăm numai datele necesare fluxurilor definite în acest laborator și precedent.

| Set de date       | Date necesare                            | Nevoia produsului                |
| ----------------- | ---------------------------------------- | -------------------------------- |
| Identitatea Stock | simbol, nume, bursă, status, categorie   | Search, Filter, Overview         |
| Latest price      | simbol, preț, variație, ora furnizorului | Stock price, Overview, Watchlist |
| Price history     | simbol, preț, momentul observației       | History                          |
| Daily summary     | open, high, low, close                   | Overview și contextul evoluției  |

### Frecvența de sincronizare — ipoteze

Pentru estimare presupunem:

- latest price: sincronizare suficient de frecventă pentru a respecta limita de vechime de 15 minute;
- price history: punct la fiecare **5 minute** în sesiunea principală;
- date de referință: actualizate zilnic;
- daily summary: o înregistrare per Stock și zi.

Acestea sunt ipoteze despre datele păstrate. Nu presupunem că fiecare cerere către Dashboard produce o cerere către Market Data Provider, conform condiției explicite a laboratorului.

---

## 4.4. Înregistrări reprezentative

Pentru estimare folosim dimensiuni medii simple.

### Date de referință Stock

| Câmp       | Exemplu        | Buget estimat | Explicație                                                 |
| ---------- | -------------- | ------------- | ---------------------------------------------------------- |
| `symbol`   | `AAPL`         | 16 B          | Buget pentru simbolul bursier                              |
| `name`     | `Apple Inc.`   | 128 B         | Buget pentru numele companiei, inclusiv denumiri mai lungi |
| `exchange` | `NASDAQ`       | 16 B          | Buget pentru denumirea sau codul bursei                    |
| `type`     | `COMMON_STOCK` | 24 B          | Buget pentru categoria instrumentului                      |
| `status`   | `ACTIVE`       | 16 B          | Buget pentru starea instrumentului                         |
| **Total**  |                | **200 B**     |                                                            |
### Latest price

| Câmp                 | Reprezentare presupusă       | Dimensiune |
| -------------------- | ---------------------------- | ---------- |
| `symbol`             | Buget text pentru simbol     | 16 B       |
| `price`              | Valoare numerică             | 8 B        |
| `change`             | Valoare numerică             | 8 B        |
| `provider_time`      | Timestamp                    | 8 B        |
| **Subtotal câmpuri** |                              | **40 B**   |
| Marjă de estimare    | Rotunjire pentru planificare | 8 B        |
| **Total estimat**    |                              | **48 B**   |

### Price history

| Câmp        | Reprezentare presupusă   | Dimensiune |
| ----------- | ------------------------ | ---------- |
| `symbol`    | Buget text pentru simbol | 16 B       |
| `price`     | Valoare numerică         | 8 B        |
| `timestamp` | Timestamp                | 8 B        |
| **Total**   |                          | **32 B**   |

### Daily summary

| Câmp                 | Reprezentare presupusă       | Dimensiune |
| -------------------- | ---------------------------- | ---------- |
| `symbol`             | Buget text pentru simbol     | 16 B       |
| `open`               | Preț de deschidere           | 8 B        |
| `high`               | Preț maxim                   | 8 B        |
| `low`                | Preț minim                   | 8 B        |
| `close`              | Preț de închidere            | 8 B        |
| `date`               | Dată calendaristică          | 4 B        |
| **Subtotal câmpuri** |                              | **52 B**   |
| Marjă de estimare    | Rotunjire pentru planificare | 12 B       |
| **Total estimat**    |                              | **64 B**   |

Valorile reprezintă dimensiuni medii de lucru pentru estimarea brută, nu dimensiuni măsurate ale unei implementări.

---

## 4.5. Price history

Sesiunea principală are 6.5 ore:

```text
6.5 × 60 = 390 minute
```

La un punct la fiecare 5 minute:

```text
390 / 5 = 78 puncte / Stock / zi
```

Păstrăm istoricul timp de **365 zile**.

Numărul de înregistrări:

```text
5,700 × 78 × 365
= 162,279,000 records
```

Stocarea:

```text
162,279,000 × 32 bytes
= 5,192,928,000 bytes
≈ 4.84 GiB
```

---

## 4.6. Calculul stocării

Formula cerută este:

```text
raw storage = record count × average bytes per record
```

iar pentru History:

```text
history record count
= supported Stocks × history points per Stock per day × retained days
```

| Set de date              | Decizia și păstrarea   | Număr de înregistrări | Bytes / înregistrare |                  Stocare brută |
| ------------------------ | ---------------------- | --------------------: | -------------------: | -----------------------------: |
| Date de referință Stock  | 5.700 Stocks active    |                 5.700 |                200 B |                    1.140.000 B |
| Cele mai recente prețuri | Un preț curent / Stock |                 5.700 |                 48 B |                      273.600 B |
| Price history            | 78 puncte/zi, 365 zile |           162.279.000 |                 32 B |                5.192.928.000 B |
| Daily summary            | 1/Stock/zi, 365 zile   |             2.080.500 |                 64 B |                  133.152.000 B |
| **Total**                |                        |                       |                      | **5.327.493.600 B ≈ 4.96 GiB** |

### Creșterea zilnică

Price history:

```text
5,700 × 78 × 32
= 14,227,200 bytes/zi
≈ 13.57 MiB/zi
```

Daily summary:

```text
5,700 × 64
= 364,800 bytes/zi
≈ 0.35 MiB/zi
```

Total:

```text
≈ 13.92 MiB/zi
```

### Stocarea inițială

După prima zi:

```text
date referință + latest price
+ o zi History
+ o zi Daily summary

≈ 15.3 MB brut
```

### Stocarea după un an

```text
≈ 5.33 GB
≈ 4.96 GiB brut
```

Estimarea este pentru **date brute**. Nu include indexuri, replici, jurnale, copii de siguranță sau alte costuri ale unei implementări.

---

# 5. Posibile blocaj

Un RPS ridicat nu dovedește existența unui blocaj. El indică o cale care trebuie investigată și măsurată.

| Calitate            | Posibil blocaj                                   | Dovezi din laborator                                                                                                                | Efect posibil                                                                                        | Ce trebuie măsurat în continuare                                                                                  |
| ------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Latență**         | Citirea Stock price                              | La 3.000 Utilizatori produce 600 din 792 RPS stabil, aproximativ 75.8% din trafic.                                                  | În perioade aglomerate, p95 poate depăși limita de 2 s.                                              | Latența p50 și p95 pentru Stock price la 600 RPS și la traficul de vârf.                                          |
| **Consistență**     | Actualizarea prețurilor și Watchlist             | Prețurile pot avea aproximativ 15 minute întârziere; după modificarea Watchlist, următoarea citire trebuie să reflecte modificarea. | Utilizatorul poate vedea un preț prea vechi sau un Watchlist care nu reflectă ultima modificare.     | Vechimea datelor afișate și timpul dintre confirmarea modificării Watchlist și vizibilitatea ei.                  |
| **Debit**           | Stock price și traficul de la deschiderea pieței | 7.920 RPS stabil la 30.000 Utilizatori și țintă de 10.296 RPS la deschidere.                                                        | Citirile pot să nu mai fie finalizate în limitele de calitate.                                       | Debit maxim de citiri acceptabile menținând p95 ≤ 2 s.                                                            |
| **Disponibilitate** | Market Data Provider                             | Stock price și History depind de datele furnizorului extern.                                                                        | Dashboard-ul poate rămâne accesibil, dar datele despre preț pot deveni întârziate sau indisponibile. | Disponibilitatea furnizorului, vechimea ultimului rezultat valid și procentul citirilor cu rezultat indisponibil. |

### Latență

`Stock price` este principalul candidat deoarece domină traficul stabil modelat.

```text
600 / 792 ≈ 75.8%
```

Dacă timpul de procesare crește odată cu traficul, cerința:

```text
p95 <= 2 secunde
```

poate fi încălcată.

Trebuie efectuată ulterior o măsurare la volumul țintă pentru a verifica ipoteza.

### Consistență

Există două stări importante:

```text
Market Data Provider -> prețul poate deveni vechi

Utilizator -> modifică Watchlist
             -> următoarea citire trebuie să vadă modificarea
```

Trebuie măsurată vechimea efectivă a datelor și timpul necesar pentru ca o modificare confirmată a Watchlist-ului să fie observabilă.

### Debit

La nivelul maxim analizat:

```text
steady = 7,920 RPS

market-open subtotal = 9,360 RPS

capacity target = 10,296 RPS
```

Următoarea proiectare trebuie să poată fi testată la aproximativ **10.296 RPS**, păstrând simultan țintele de latență și consistență.

### Disponibilitate

Market Data Provider este o dependență externă deja identificată în System Context din Lab 1.

Dacă acesta nu furnizează un rezultat valid, Dashboard-ul nu trebuie să transforme lipsa datelor într-un succes fals.

Rezultatul pentru utilizator trebuie să rămână explicit:

```text
date acceptate -> afișează prețul și ora furnizorului

date vechi -> marchează întârzierea

date lipsă/neacceptate -> indisponibil
```

Astfel, defectarea furnizorului poate afecta disponibilitatea informațiilor despre piață fără ca Dashboard-ul să prezinte date incorecte ca fiind actuale.

---

## Listă de verificare

- [x] Au fost definite cerințe măsurabile pentru latență, disponibilitate, consistență și debit.
- [x] Au fost definite și justificate ținte separate de disponibilitate pentru orele de tranzacționare și restul zilei.
- [x] A fost calculat bugetul de indisponibilitate pentru un interval de 30 zile.
- [x] Au fost calculate toate cele șase citiri pentru 300, 3.000 și 30.000 Utilizatori.
- [x] Au fost calculate RPS stabil și RPS la deschiderea pieței.
- [x] Marja de 10% a fost aplicată după subtotal, iar numai ținta finală a fost rotunjită.
- [x] A fost cercetat și definit domeniul Stock pentru prima versiune.
- [x] Au fost definite datele necesare pentru Search, Filter, Overview, Stock price și History.
- [x] A fost estimată stocarea brută inițială, zilnică și pentru un an.
- [x] Au fost precizate ipotezele, unitățile și perioadele de păstrare.
- [x] A fost analizat câte un posibil blocaj pentru fiecare calitate principală.
- [x] Nu a fost selectată încă o arhitectură internă, bază de date, cache, coadă sau tehnologie.

---

## Surse

- [Investor.gov — Stocks FAQ](https://www.investor.gov/introduction-investing/investing-basics/investment-products/stocks).
- [Nasdaq U.S. Equities](https://www.nasdaq.com/products/north-american-markets/us-equities).
- [NYSE Listings](https://www.nyse.com/listings/why-nyse).
