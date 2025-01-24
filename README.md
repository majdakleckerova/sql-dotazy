# SQL 
Pro  předměty **KI/URD** a **KI/RDBS**

```sql
CREATE INDEX idx_suroviny ON public."Suroviny" USING GIN(to_tsvector('english', nazev_suroviny));
```

```sql
SELECT *
FROM public."Suroviny"
WHERE to_tsvector('english', nazev_suroviny) @@ to_tsquery('english','pomeranč');
```

