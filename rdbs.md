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
### d) FUNCTION (1x)
- která bude realizovat výpočet nějaké hodnoty z dat v DB

	**Výpočet profitu na konkrétním nápoji**
```sql
CREATE OR REPLACE FUNCTION public."profit"(zvoleny_napoj VARCHAR)
RETURNS money AS $$
DECLARE
    profit money;
BEGIN
    SELECT 
        (public."Napoje".cena_napoje - SUM(public."Suroviny".cena_za_ml * public."Recepty".mnozstvi_ml))::money INTO profit
    FROM 
        public."Napoje"
    JOIN 
        public."Recepty" ON public."Napoje".id_napoje = public."Recepty".id_napoje
    JOIN 
        public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny
    WHERE 
        public."Napoje".nazev_napoje = zvoleny_napoj
    GROUP BY 
        public."Napoje".nazev_napoje, public."Napoje".cena_napoje;

    RETURN profit;
END; 
$$ LANGUAGE plpgsql;
```

```sql
SELECT public."profit"('Mojito');
```

### e) PROCEDURE (1x)
- která bude používat **1x CURSOR** a také **1x ošetření chyb** (HANDLER /
TRY…CATCH / RAISE / EXCEPTION - dle zvoleného DBMS)
- např. vytvoří a naplní novou tabulku informacemi o náhodných slevách na vybrané
výrobky, nebo zákazníkům vygeneruje slevové bonusy podle určitých podmínek, apod.

	- vytvoření view **Profity_napoju**
   
```sql
CREATE OR REPLACE VIEW public."Profity_Napoju" AS
SELECT 
    public."Napoje".nazev_napoje,
    public."Napoje".cena_napoje::money AS cena_napoje,
    SUM(public."Suroviny".cena_za_ml * public."Recepty".mnozstvi_ml)::money AS naklady_na_napoj,
    (public."Napoje".cena_napoje - SUM(public."Suroviny".cena_za_ml * public."Recepty".mnozstvi_ml))::money AS profit
FROM 
    public."Napoje"
JOIN 
    public."Recepty" ON public."Napoje".id_napoje = public."Recepty".id_napoje
JOIN 
    public."Suroviny" ON public."Recepty".id_suroviny = public."Suroviny".id_suroviny
GROUP BY 
    public."Napoje".nazev_napoje, public."Napoje".cena_napoje;
```

	- vytvoření tabulky **SlevyNaNapojich**
```sql
CREATE TABLE public."SlevyNaNapojich" (
    id SERIAL PRIMARY KEY,
    nazev_napoje VARCHAR NOT NULL,
    puvodni_cena MONEY,
    sleva_percent NUMERIC,
    nova_cena MONEY
);
```

	- procedura **SlevyNaNejziskovejsiNapoje** , dávající top 10 nejziskovějším nápojům slevu 10%
 ```sql
CREATE OR REPLACE PROCEDURE public."SlevyNaNejziskovejsiNapoje"()
LANGUAGE plpgsql
AS $$
DECLARE
    napoj_cursor CURSOR FOR 
        SELECT nazev_napoje, cena_napoje, profit 
        FROM public."Profity_Napoju"
        ORDER BY profit DESC
        LIMIT 10;
    record RECORD;
    nova_cena MONEY;
BEGIN
    OPEN napoj_cursor;

    LOOP
        FETCH napoj_cursor INTO record;
        EXIT WHEN NOT FOUND;

        nova_cena := record.cena_napoje * 0.9;

        BEGIN
            INSERT INTO public."SlevyNaNapojich" (nazev_napoje, puvodni_cena, sleva_percent, nova_cena)
            VALUES (record.nazev_napoje, record.cena_napoje, 10, nova_cena);
        EXCEPTION
            WHEN OTHERS THEN
                RAISE NOTICE 'Chyba při vkládání slevy pro nápoj %', record.nazev_napoje;
        END;
    END LOOP;

    CLOSE napoj_cursor;
END;
$$;
```
		- volání procedury:
```sql
CALL public."SlevyNaNejziskovejsiNapoje"();
```
