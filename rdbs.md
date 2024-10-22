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
