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
```sql
SELECT public."Napoje".id_napoje, public."Napoje".nazev_napoje, 
       (SELECT COUNT(*) 
        FROM public."Polozky_objednavky" 
        WHERE public."Polozky_objednavky".id_napoje = public."Napoje".id_napoje) AS pocet_objednavek
FROM public."Napoje"
ORDER BY pocet_objednavek DESC;
```

- jeden SELECT bude obsahovat nějakou **analytickou funkci** (SUM, COUNT, AVG,…) spolu
s agregační klauzulí GROUP BY
- jeden SELECT bude řešit **rekurzi** nebo hierarchii (SELF JOIN)
