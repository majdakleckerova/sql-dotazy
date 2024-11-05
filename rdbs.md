# ZÁPOČET
## Připravené příkazy nad DB
### a) SELECT (4x)
- jeden SELECT vypočte **průměrný počet záznamů** na jednu tabulku v DB
```sql
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

- jeden SELECT bude obsahovat **vnořený SELECT**
  	... *Kolikrát byl každý nápoj objednán*
```sql
SELECT public."Napoje".id_napoje, public."Napoje".nazev_napoje, 
       (SELECT COUNT(*) 
        FROM public."Polozky_objednavky" 
        WHERE public."Polozky_objednavky".id_napoje = public."Napoje".id_napoje) AS pocet_objednavek
FROM public."Napoje"
ORDER BY pocet_objednavek DESC;
```

- jeden SELECT bude obsahovat nějakou **analytickou funkci** (SUM, COUNT, AVG,…) spolu s agregační klauzulí GROUP BY
	... *průměrná cena nápoje pro každý typ nápoje*
```sql
SELECT public."Typy_napoju".nazev_typu,
	AVG(public."Napoje".cena_napoje::numeric) AS prumerna_cena
FROM public."Napoje" 
	JOIN public."Typy_napoju" 
	ON public."Napoje".id_typ_napoje = public."Typy_napoju".id_typ_napoje
	GROUP BY public."Typy_napoju".nazev_typu
	ORDER BY prumerna_cena;
```


- jeden SELECT bude řešit **rekurzi** nebo hierarchii (SELF JOIN)
  	... *Vrací pozice a jejich nadřízené pozice*
```sql
  SELECT p1.id_pozice AS pozice_id, 
       p1.nazev_pozice AS nazev_pozice, 
       p2.id_pozice AS id_nadrizene_pozice, 
       p2.nazev_pozice AS nadrizena_pozice
FROM public."Pozice" p1
LEFT JOIN public."Pozice" p2
ON p1.id_nadpozice = p2.id_pozice
ORDER BY p1.id_pozice;
```

### b) VIEW (1X)
s podstatnými informacemi z několika tabulek najednou
- alespoň **tři** tabulky, mezi informacemi nemají figurovat FK a nedůležité informace
- pro spojení tabulek použijte různé typy příkazu **JOIN** (inner, left, right, natural, …)

```sql
CREATE VIEW public."Recepty_na_koktejly" AS
SELECT
    public."Napoje".nazev_napoje,
    public."Typy_napoju".druh_napoje, 
    public."Suroviny".nazev_suroviny,
    public."Recepty".mnozstvi_ml
FROM
    public."Napoje"
    INNER JOIN public."Typy_napoju" ON public."Napoje".id_typ_napoje = public."Typy_napoju".id_typ_napoje 
	LEFT JOIN public."Recepty" ON public."Napoje".id_napoje = public."Recepty".id_napoje
    NATURAL JOIN public."Suroviny" 
WHERE
    public."Typy_napoju".druh_napoje IN ('alkoholický','nealkoholický'); 
```
```sql
SELECT * FROM public."Recepty_na_koktejly"
```

### c) INDEX (1X)
indexový soubor nad nějakým sloupcem tabulky
- alespoň jeden **netriviální** indexový soubor (unikátní, fulltextový, …)
  
```sql
CREATE UNIQUE INDEX idx_unikat_telefon ON public."Pracovnici"(telefonni_cislo);
```
*použití - přidání dalšího pracovníka se stejným teleofnním číslem*
```sql
INSERT INTO public."Pracovnici" 
(id_pracovnika, jmeno, prijmeni, datum_narozeni, adresa, telefonni_cislo, uvazek, cislo_uctu, datum_nastupu, id_pozice) 
VALUES 
(022,'Petr', 'Černý', '1985-05-15', 'Nějaká ulice 123, Praha', '717133951', 'DPP', '123456789/0800', '2023-01-01', 1);
```
```sql
CHYBA: Key (telefonni_cislo)=(717133951) already exists.duplicate key value violates unique constraint "idx_unikat_telefon" 
```
