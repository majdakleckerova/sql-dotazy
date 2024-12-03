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
	UNION ALL
	SELECT COUNT(*) FROM public."Tydenni_vyplaty"
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
	ORDER BY prumerna_cena DESC;
```


- jeden SELECT bude řešit **rekurzi** nebo hierarchii (SELF JOIN)
  	... *Vrací pozice a jejich nadřízené pozice*
```sql
SELECT p1.id_pozice AS id_pozice, 
       p1.nazev_pozice AS nazev_pozice, 
       p2.id_pozice AS id_nadrizene_pozice, 
       p2.nazev_pozice AS nazev_nadrizene_pozice,
       pr1.id_pracovnika AS id_pracovnika_nadrizene_pozice,
       CONCAT(pr1.jmeno, ' ', pr1.prijmeni) AS jmeno_prijmeni_nadrizene_pozice
FROM public."Pozice" p1
LEFT JOIN public."Pozice" p2
  ON p1.id_nadpozice = p2.id_pozice
LEFT JOIN public."Pracovnici" pr1
  ON pr1.id_pozice = p2.id_pozice
WHERE p2.id_pozice IN (3, 4, 5)
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
CREATE UNIQUE INDEX idx_unikat_telefon ON public."Pracovnici"(telefoni_cislo);
```
*použití - přidání dalšího pracovníka se stejným teleofnním číslem*
```sql
INSERT INTO public."Pracovnici" 
(id_pracovnika, jmeno, prijmeni, datum_narozeni, adresa, telefoni_cislo, uvazek, cislo_uctu, datum_nastupu, id_pozice) 
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

### f) TRIGGER (1x)
- který ošetří práci uživatele s daty DB
- využívají se hlavně pro příkazy INSERT, **UPDATE**, DELETE nad nějakou tabulkou, ale můžete zkusit i jiný typ triggeru
- např. pro UPDATE do nové tabulky zapíše datum, čas a informace o uživateli, který nějakým způsobem upravoval data v dané tabulce

  - nová tabulka **LogZmenyCislaUctu**
```sql
CREATE TABLE public."LogZmenyCislaUctu" (
    id_log SERIAL PRIMARY KEY,
    id_pracovnika INT,
    datum_zmeny TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    puvodni_cislo_uctu VARCHAR,
    nove_cislo_uctu VARCHAR,
    uzivatel VARCHAR
);
```

- nová funkce **LogujZmenyCislaUctu** 
```sql
CREATE OR REPLACE FUNCTION public."LogujZmenyCislaUctu"()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public."LogZmenyCislaUctu" (
        id_pracovnika,
        datum_zmeny,
        puvodni_cislo_uctu,
        nove_cislo_uctu,
        uzivatel
    )
    VALUES (
        OLD.id_pracovnika,
        CURRENT_TIMESTAMP,
        OLD.cislo_uctu,
        NEW.cislo_uctu,
        current_user
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
- trigger **trigger_log_zmen_cisla_uctu**
```sql
CREATE TRIGGER trigger_log_zmen_cisla_uctu
AFTER UPDATE OF cislo_uctu ON public."Pracovnici"
FOR EACH ROW
EXECUTE FUNCTION public."LogujZmenyCislaUctu"();
```

- **zkouška**
```sql
UPDATE public."Pracovnici"
SET cislo_uctu = '1234567890'
WHERE id_pracovnika = 1;
```
```sql
SELECT * FROM public."LogZmenyCislaUctu";
```

### g) TRANSACTION (1x)
- použít v některé z předchozích procedur / funkcí
- tj. uzavřít skupinu příkazů do transakce a ošetřit případ, kdy není možné všechny uvedené
příkazy vykonat najednou (ROLLBACK)
- např. převod peněz z jednoho účtu na druhý uzavřít do transakce + ošetřit situaci kdy
odesilatel nemá na účtu dostatek financí na provedení převodu
- START/BEGIN TRANSACTION, COMMIT, ROLLBACK (případně i SAVEPOINT)

```sql
CREATE TABLE public."Tydenni_vyplaty" (
    id_pracovnika INT,
    jmeno VARCHAR,
    prijmeni VARCHAR,
    vyplata MONEY,
    zacatek_tydne DATE,
    konec_tydne DATE
);
```

```sql
CREATE OR REPLACE PROCEDURE public."VypocetVyplat"(zacatek_tydne DATE, konec_tydne DATE)
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
    start_of_week DATE;
    end_of_week DATE;
BEGIN
    -- Cyklus pro číšníky a barmany
    FOR rec IN 
        SELECT 
            public."Pracovnici".id_pracovnika, 
            public."Pracovnici".jmeno,
            public."Pracovnici".prijmeni,
            public."Pozice".mzda, 
            SUM(public."Rozpisy_smen".pocet_hodin) AS celkove_hodiny,
            DATE_TRUNC('week', public."Rozpisy_smen".datum_smeny) AS start_of_week, 
            DATE_TRUNC('week', public."Rozpisy_smen".datum_smeny) + INTERVAL '6 days' AS end_of_week
        FROM 
            public."Pracovnici"
        LEFT JOIN 
            public."Rozpisy_smen" ON public."Pracovnici".id_pracovnika IN (public."Rozpisy_smen".id_cisnika, public."Rozpisy_smen".id_barmana)
        LEFT JOIN 
            public."Pozice" ON public."Pracovnici".id_pozice = public."Pozice".id_pozice
        WHERE 
            public."Rozpisy_smen".datum_smeny >= zacatek_tydne
            AND public."Rozpisy_smen".datum_smeny <= konec_tydne
        GROUP BY 
            public."Pracovnici".id_pracovnika, 
            public."Pracovnici".jmeno, 
            public."Pracovnici".prijmeni, 
            public."Pozice".mzda, 
            DATE_TRUNC('week', public."Rozpisy_smen".datum_smeny)
    LOOP
        INSERT INTO public."Tydenni_vyplaty" (id_pracovnika, jmeno, prijmeni, vyplata, zacatek_tydne, konec_tydne, pocet_hodin)
        VALUES (rec.id_pracovnika, rec.jmeno, rec.prijmeni, rec.mzda * rec.celkove_hodiny, rec.start_of_week, rec.end_of_week, rec.celkove_hodiny);
    END LOOP;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Chyba při výpočtu výplat: %', SQLERRM;
        ROLLBACK;
END;
$$;
```
```sql
CALL public."VypocetVyplat"('2024-04-01', '2024-04-07');
```

### h) USER 
- mít předem připravené příkazy na ukázku práce s účty uživatelů
1.  umět vytvořit/odstranit účet uživatele **CREATE/DROP USER**
```sql
CREATE USER majda WITH PASSWORD 'majda123';
```
```sql
DROP USER majda;
```
2. umět se přihlásit jako právě vytvořený uživatel a ověřit dostupnost databází z pohledu nového uživatele
   **Servers** -> **Registrovat** -> **Server...** -> *Název, uživatelské jméno, heslo, údržbová databáze...*
   
3. umět vytvořit/odstranit roli CREATE/DROP ROLE (některé DBMS nemají role)
```sql
CREATE ROLE vytvoreni_databaze WITH LOGIN PASSWORD 'role123' CREATEDB;
```
```sql
CREATE ROLE role WITH LOGIN PASSWORD 'role123';
```

```sql
DROP ROLE vytvoreni_databaze;
```

5. umět přidělit/odebrat uživateli nebo roli nějaká práva GRANT / REVOKE

```sql
GRANT CONNECT ON DATABASE "URDzapocet_bar" TO majda;
```
```sql
GRANT CREATE ON SCHEMA public TO majda;
```

```sql
GRANT SELECT, INSERT, UPDATE ON public."Tydenni_vyplaty" TO majda;
```
```sql
GRANT SELECT,INSERT ON public."Napoje", public."Recepty", public."Suroviny" TO role;
```
```sql
REVOKE INSERT ON public."Napoje", public."Recepty", public."Suroviny" FROM role;
```

```sql
GRANT role TO majda;
```
```sql
REVOKE role FROM majda;
```
```sql
GRANT SELECT, UPDATE, DELETE, INSERT ON public."LogZmenyCislaUctu" to majda;
```
```sql
GRANT USAGE, SELECT ON SEQUENCE public."LogZmenyCislaUctu_id_log_seq" TO majda;
```

### i) LOCK
- mít předem připravené příkazy na ukázku zamykání tabulek
- umět zamknout/odemknout tabulku (případně celou databázi, nebo jen řádek - pokud to 
zvolený DBMS umožňuje)

- zamknutí tabulky
```sql
BEGIN;

LOCK TABLE public."Pracovnici" IN ACCESS EXCLUSIVE MODE;

UPDATE public."Pracovnici"
SET prijmeni = 'Marková'
WHERE id_pracovnika = 5;
SELECT * FROM pg_locks
COMMIT;
```
- zamknutí řádku
```sql
BEGIN;

SELECT * 
FROM public."Pracovnici"
WHERE id_pracovnika = 5
FOR UPDATE;

UPDATE public."Pracovnici"
SET prijmeni = 'Marková 2'
WHERE id_pracovnika = 5;

COMMIT;
```
### j) ORM - sqlalchemy
```sql
GRANT SELECT ON TABLE public."Stoly" TO majda;
```

```python
from sqlalchemy import create_engine, func
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy import desc

engine = create_engine('postgresql://majda:majda123@localhost:5432/URDzapocet_bar')

Base = automap_base()
Base.prepare(engine, reflect=True)

Session = sessionmaker(bind=engine)
session = Session()

stoly = Base.classes.Stoly
subquery = session.query(func.avg(stoly.kapacita_stolu)).scalar()
result = session.query(func.count(stoly.nazev_stolu)).filter(stoly.kapacita_stolu > subquery).scalar()
print(f"\nPrůměrná kapacita stolu: {subquery}\nPočet stolů s kapacitou větší než průměr: {result}; to jest:")

stoly_vetsi_nez_prumer = session.query(stoly).filter(stoly.kapacita_stolu > subquery).order_by(desc(stoly.kapacita_stolu)).all()
for stul in stoly_vetsi_nez_prumer:
    print(f"Stůl ID: {stul.id_stolu}, Název stolu: {stul.nazev_stolu}, Kapacita: {stul.kapacita_stolu}")
```


