# URD - příkazy do databáze 

## Implementace databáze

### VYTVOŘENÍ TABULKY
``` {SQL}
CREATE TABLE public."Napoje" (
    id_napoje integer PRIMARY KEY,
    nazev_napoje text,
    objem_napoje numeric,
    cena_napoje money,
);
```
### PŘIDÁNÍ CIZÍHO KLÍČE DO TABULKY
``` {SQL}
ALTER TABLE public."Napoje" ADD CONSTRAINT id_typ_napoje FOREIGN KEY (id_typ_napoje) REFERENCES public."Typy_napoju"(id_typ_napoje);
```

### PLNĚNÍ DATY - IMPORT Z CSV SOUBORU , UTF-8
``` {SQL}
COPY public."Napoje" FROM 'C:\majda LS\URD\Napoje_2.csv' DELIMITER ';' CSV HEADER;
```

### PŘIDÁNÍ ŘÁDKU 
``` {SQL}
INSERT INTO public."Recepty" (mnozstvi_ml, id_suroviny, id_napoje)
VALUES (400,046,049);
```

### FILTROVÁNÍ DAT Z DATABÁZE
``` {SQL}
SELECT * FROM public."Napoje" WHERE id_typ_napoje = '03';
```
``` {SQL}
SELECT * FROM public."Napoje" WHERE cena_napoje <= 100::money ;
``` 
## Příkazy 

## 0. Průměrný počet záznamů na tabulku
``` {SQL}
SELECT SUM(cnt) / COUNT(*) AS prumer_zaznamu_na_tabulku
FROM (
    SELECT COUNT(*) AS cnt FROM public."Napoje"
    UNION ALL
    SELECT COUNT(*) FROM public."Objednavky"
	UNION ALL	
	SELECT COUNT(*) FROM public."Polozky_objednavky"
	UNION ALL
	SELECT COUNT(*) FROM public."Pozice"
	UNION ALL
	SELECT COUNT(*) FROM public."Pracovnici"
	UNION ALL
	SELECT COUNT(*) FROM public."Recepty"
	UNION ALL
	SELECT COUNT(*) FROM public."Rezervace"
	UNION ALL
	SELECT COUNT(*) FROM public."Rozpisy_smen"
	UNION ALL
	SELECT COUNT(*) FROM public."Stoly"
	UNION ALL
	SELECT COUNT(*) FROM public."Suroviny"
	UNION ALL
	SELECT COUNT(*) FROM public."Typy_napoju"
) AS counts;
```

### Výstup
počet tabulek | počet záznamů | průměrný počet záznamů na tabulku
--- | --- | ---
11 | 455| 41.3636364

## 1. 5 NejaktivnějšíCH barmanŮ :) (nejvíc směn)

``` {SQL}
SELECT public."Pracovnici".id_pracovnika, public."Pracovnici".jmeno, public."Pracovnici".prijmeni, COUNT(public."Rozpisy_smen".id_smeny) AS pocet_smen
FROM public."Pracovnici" JOIN public."Rozpisy_smen" ON public."Pracovnici".id_pracovnika = public."Rozpisy_smen".id_barmana
WHERE public."Pracovnici".id_pozice = 02
GROUP BY public."Pracovnici".id_pracovnika, public."Pracovnici".jmeno, public."Pracovnici".prijmeni
ORDER BY COUNT(public."Rozpisy_smen".id_smeny) DESC;
```

## 2. 5 nejoblíbenějších nápojů
``` {SQL}
SELECT public."Napoje".id_napoje, public."Napoje".nazev_napoje, SUM(public."Polozky_objednavky".pocet_kusu) AS celkem_prodano
FROM public."Napoje"
JOIN public."Polozky_objednavky" ON public."Napoje".id_napoje = public."Polozky_objednavky".id_napoje
GROUP BY public."Napoje".id_napoje, public."Napoje".nazev_napoje
ORDER BY SUM(public."Polozky_objednavky".pocet_kusu) DESC
LIMIT 5;
```

## 3. Nejvýdělečnější pracovníci
``` {SQL}
SELECT
    public."Pracovnici".jmeno,
    public."Pracovnici".prijmeni,
    public."Pozice".nazev_pozice,
    SUM(public."Rozpisy_smen".pocet_hodin) AS pocet_odpracovanych_hodin,
    SUM(public."Rozpisy_smen".pocet_hodin * public."Pozice".mzda) AS vydelek
FROM
    public."Pracovnici"
JOIN
    public."Rozpisy_smen" ON public."Pracovnici".id_pracovnika = public."Rozpisy_smen".id_cisnika OR public."Pracovnici".id_pracovnika = public."Rozpisy_smen".id_barmana
JOIN
    public."Pozice" ON public."Pracovnici".id_pozice = public."Pozice".id_pozice
GROUP BY
    public."Pracovnici".id_pracovnika,
    public."Pozice".nazev_pozice
ORDER BY
    vydelek DESC;
```

## 4. Nejziskovější nápoje a jejich typ
``` {SQL}
SELECT
    public."Napoje".nazev_napoje,
    public."Typy_napoju".nazev_typu,
    (public."Napoje".cena_napoje - SUM(public."Recepty".mnozstvi_ml * public."Suroviny".cena_za_ml)) AS vydelek_na_jedno
FROM
    public."Napoje"
JOIN
    public."Recepty" ON public."Napoje".id_napoje = public."Recepty".id_napoje
JOIN
    public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny
JOIN
    public."Typy_napoju" ON public."Napoje".id_typ_napoje = public."Typy_napoju".id_typ_napoje
GROUP BY
    public."Napoje".nazev_napoje,
    public."Typy_napoju".nazev_typu,
    public."Napoje".cena_napoje
ORDER BY
    vydelek_na_jedno DESC;
```

## 5. Došel ananasovej džus. Vyfiltruj všechny názvy nápojů, co maj v receptech ananasovej džus
``` {SQL}
SELECT public."Napoje".nazev_napoje
FROM public."Recepty"
JOIN public."Napoje" ON public."Recepty".id_napoje = public."Napoje".id_napoje
JOIN public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny
WHERE public."Suroviny".id_suroviny = 027;
```

## 6. Nejvěrnější zákazník podle počtu rezervací
``` {SQL}
SELECT public."Rezervace".jmeno_rezervujiciho, COUNT(*) AS pocet_rezervaci
FROM public."Rezervace"
GROUP BY public."Rezervace".jmeno_rezervujiciho
ORDER BY pocet_rezervaci DESC
LIMIT 2;
```

## 7. Seznam číšníků, co vyřídili více než 3 objednávky
``` {SQL}
SELECT public."Pracovnici".jmeno, public."Pracovnici".prijmeni, public."Objednavky".id_cisnika, COUNT(public."Objednavky".id_objednavky) AS pocet_objednavek
FROM public."Pracovnici"
JOIN public."Objednavky" ON public."Pracovnici".id_pracovnika = public."Objednavky".id_cisnika
GROUP BY public."Pracovnici".id_pracovnika, public."Pracovnici".jmeno, public."Pracovnici".prijmeni, public."Objednavky".id_cisnika
HAVING COUNT(public."Objednavky".id_objednavky) > 3
ORDER BY COUNT(public."Objednavky".id_objednavky) DESC;
```

## 8. Zobrazení nejprodávanějšího nápoje od každého typu (pivo, víno, koktejl, ...)

- vytvoření pohledu (CREATE VIEW)
``` {SQL}
CREATE VIEW public."nejprodavanejsi_napoje" AS
SELECT 
    public."Typy_napoju".nazev_typu,
    public."Napoje".nazev_napoje,
    SUM(public."Polozky_objednavky".pocet_kusu) AS pocet_prodanych_kusu
FROM 
    public."Typy_napoju"
JOIN 
    public."Napoje" ON public."Typy_napoju".id_typ_napoje = public."Napoje".id_typ_napoje
JOIN 
    public."Polozky_objednavky" ON public."Napoje".id_napoje = public."Polozky_objednavky".id_napoje
GROUP BY 
    public."Typy_napoju".id_typ_napoje, public."Typy_napoju".nazev_typu, public."Napoje".nazev_napoje;
```

- aplikace pohledu
``` {SQL}
SELECT
    nazev_typu,
    nazev_napoje,
    pocet_prodanych_kusu
FROM
    public."nejprodavanejsi_napoje"
WHERE
    (nazev_typu, pocet_prodanych_kusu) IN (
        SELECT
            nazev_typu,
            MAX(pocet_prodanych_kusu)
        FROM
            public."nejprodavanejsi_napoje"
        GROUP BY
            nazev_typu
    );
```

## 9. Recept na Mojito (JOIN 3 tabulek)
``` {SQL}
SELECT
    public."Suroviny".nazev_suroviny,
    public."Recepty".mnozstvi_ml
FROM
    public."Recepty"
JOIN
    public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny
JOIN
    public."Napoje" ON public."Recepty".id_napoje = public."Napoje".id_napoje
WHERE
    public."Napoje".nazev_napoje = 'Mojito';
```

## 10. Směny s největšími tržbami od 3. května

- vytvoření pohledu "Celkove_trzby" (CREATE VIEW)

``` {SQL}
CREATE VIEW public."Celkove_trzby_smen_od_3_kvetna" AS
SELECT
    public."Rozpisy_smen".id_smeny,
    SUM(public."Polozky_objednavky".pocet_kusu * public."Napoje".cena_napoje) AS celkove_trzby
FROM
    public."Rozpisy_smen"
LEFT JOIN
    public."Objednavky" ON public."Objednavky".datum_objednavky::date = public."Rozpisy_smen".datum_smeny::date
LEFT JOIN
    public."Polozky_objednavky" ON public."Objednavky".id_objednavky = public."Polozky_objednavky".id_objednavky
LEFT JOIN
    public."Napoje" ON public."Polozky_objednavky".id_napoje = public."Napoje".id_napoje
WHERE
    public."Rozpisy_smen".datum_smeny >= '2024-05-03'::date
GROUP BY
    public."Rozpisy_smen".id_smeny;
```

- seřazení, JOIN s datumem směny a příjmeními barmana a číšníka na dané směně
``` {SQL}
SELECT
    public."Rozpisy_smen".datum_smeny,
    B.prijmeni AS prijmeni_barmana,
    C.prijmeni AS prijmeni_cisnika,
    public."Celkove_trzby_smen_od_3_kvetna".celkove_trzby
FROM
    public."Celkove_trzby_smen_od_3_kvetna" 
JOIN
    public."Rozpisy_smen" ON public."Celkove_trzby_smen_od_3_kvetna".id_smeny = public."Rozpisy_smen".id_smeny
JOIN
    public."Pracovnici" AS B ON public."Rozpisy_smen".id_barmana = B.id_pracovnika
JOIN
    public."Pracovnici" AS C ON public."Rozpisy_smen".id_cisnika = C.id_pracovnika
ORDER BY
    public."Celkove_trzby_smen_od_3_kvetna".celkove_trzby DESC;
```


## 11. Seznam receptů: Spojení tabulek Napoje, Suroviny a Recepty (LEFT JOIN) -> zobrazení nazev_napoje , mnozstvi_ml, nazev_suroviny
``` {SQL}
SELECT
    public."Napoje".nazev_napoje,
    public."Recepty".mnozstvi_ml,
    public."Suroviny".nazev_suroviny
FROM
    public."Napoje"
LEFT JOIN
    public."Recepty" ON public."Napoje".id_napoje = public."Recepty".id_napoje
LEFT JOIN
    public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny;
```

