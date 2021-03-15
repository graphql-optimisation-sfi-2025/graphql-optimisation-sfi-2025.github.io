<h1>LEVEL UP!</h1>

::::
<img src="img/ruby-levels.jpg" height=600></img>
<img src="img/sql-levels.jpg" height=600 class='fragment'></img>
<img src="img/sql-levels2.jpg" height=600 class='fragment'></img>

::::
<!-- smutny, rozczarowany kotek -->
<section
  data-background-image="img/sadkitten-meme.jpg"
  style='min-height=100% important!'
>

::::

<section
  data-background-image="img/sadkitten.jpg"
  style='min-height=100% important!'
>

<div class='image-overlay' >
  <h2>Dlaczego baza danych nie używa mojego indeksu?</h2>
  <p style='padding: 150px'></p>
  <h3> Maciek Rząsa, <a href='https://twitter.com/mjrzasa'>@mjrzasa</a> </h3>
  <h4 >Rzeszów Ruby User Group, 8.10.2019 </h4>
</div>

</section>



::::

<img src="img/disk.jpg" height=600></img>

:::

<img src="img/latency.png" height=700></img>

::::

<img src="img/fig01_01_index_leaf_nodes.en.MMHwYDFb.png" height=600></img>

:::

<img src="img/fig01_02_tree_structure.en.BdEzalqw.png" height=600></img>

:::

<img src="img/fig01_03_tree_traversal.en.niC7Q5jq.png" height=600></img>

:::

## Złożone indeksy


```
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE date_of_birth >= TO_DATE(?, 'YYYY-MM-DD')
   AND date_of_birth <= TO_DATE(?, 'YYYY-MM-DD')
   AND subsidiary_id = ?

```


```
CREATE INDEX employee_dob
    ON employees (subsidiary_id, date_of_birth)

# vs

CREATE INDEX employee_dob
    ON employees (date_of_birth, subsidiary_id)
```
:::

```
CREATE INDEX employee_dob
    ON employees (subsidiary_id, date_of_birth)

QUERY PLAN
-------------------------------------------------------------------
Index Scan using emp_test on employees
  (cost=0.01..8.59 rows=1 width=16)
  Index Cond: (date_of_birth >= to_date('1971-01-01','YYYY-MM-DD'))
      AND (date_of_birth <= to_date('1971-01-10','YYYY-MM-DD'))
      AND (subsidiary_id = 27::numeric)
```

```
CREATE INDEX employee_dob
    ON employees (date_of_birth, subsidiary_id)

                            QUERY PLAN
-------------------------------------------------------------------
Index Scan using emp_test on employees
   (cost=0.01..8.29 rows=1 width=17)
   Index Cond: (subsidiary_id = 27::numeric)
       AND (date_of_birth >= to_date('1971-01-01', 'YYYY-MM-DD'))
       AND (date_of_birth <= to_date('1971-01-10', 'YYYY-MM-DD'))
```

:::

<img src="img/fig02_02_range_scan.en.lPp+MnUS.png" height=600></img>
<img src="img/fig02_03_range_scan.en.bsqXV98T.png" height=600></img>

:::

## Dlaczego baza nie używa mojego indeksu?

<h2 class='answer'> Warunek z zakresem dotyczy początkowej kolumny w indeksie </h2>

:::

```
SELECT first_name, last_name, date_of_birth
  FROM employees WHERE subsidiary_id = ?
```

<img src="img/fig02_02_range_scan.en.lPp+MnUS.png" height=600></img>
<img src="img/fig02_03_range_scan.en.bsqXV98T.png" height=600></img>

:::

## Dlaczego baza nie używa mojego indeksu?

<h2 class='answer'> Warunki nie dotyczą pierwszych kolumn indeksie </h2>
```
CREATE INDEX employee_dob
    ON employees (date_of_birth, subsidiary_id);

SELECT * FROM employees WHERE subsidiary_id = 1;
```
:::

## Funkcje

:::

```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE UPPER(last_name) = UPPER('stefan')

```

```
CREATE INDEX emp_up_name
    ON employees (last_name)
```

```
                     QUERY PLAN
------------------------------------------------------
 Seq Scan on employees
   (cost=0.00..1722.00 rows=50 width=17)
   Filter: (upper((last_name)::text) = 'STEFAN'::text)

```

:::
```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE UPPER(last_name) = UPPER('stefan')

```
```
CREATE INDEX emp_up_name
    ON employees (UPPER(last_name))
```
```
                      QUERY PLAN
----------------------------------------------------------
 Index Scan using emp_up_name on employees
   (cost=0.00..8.28 rows=1 width=17)
   Index Cond: (upper((last_name)::text) = 'STEFAN'::text

```

:::

### Podobnie w przypadku

```
ADDTIME(date_column, time_column)
TO_NUMBER(numeric_string)
TO_CHAR(numeric_column)
numeric_column*2 > 100
```

:::
## Dlaczego baza nie używa mojego indeksu?

<h2 class='answer'> Indeks jest na kolumnie, a zapytanie na wyrażeniu </h2>
```
SELECT first_name, last_name, phone_number
  FROM employees
 WHERE UPPER(last_name) = UPPER('stefan')

```
```
CREATE INDEX emp_up_name
    ON employees (last_name)
```
:::

## Statystyki i indeksy częściowe
```
VACUUM ANALYZE
ANALYZE
```

```
SELECT * FROM messages WHERE processed = true
SELECT * FROM messages WHERE processed = false
```

```
CREATE INDEX messages_processed
  ON messages (processed)
```
<div class='fragment'>
<div></div>

```
CREATE INDEX messages_not_processed
  ON messages (processed)
  WHERE processed = false
```
</div>
:::
## Dlaczego baza nie używa mojego indeksu?

<h2 class='answer'> Wyszukanie zwraca zbyt wiele danych </h2>
<h2 class='answer'> Query optimizer _myśli_, że zapytanie zwraca zbyt wiele danych </h2>

:::

## Rails
```ruby
add_index(:suppliers, :name)
# CREATE INDEX suppliers_name_index ON suppliers(name)

add_index(:accounts, [:branch_id, :party_id])
#CREATE INDEX accounts_branch_id_party_id_index
#  ON accounts(branch_id, party_id)
```

```
User.where('created_at < ?', 1.day.ago).explain

```
:::

## Dlaczego baza nie używa mojego indeksu?

* Warunek z zakresem dotyczy początkowej kolumny w indeksie
* Warunki nie dotyczą pierwszych kolumn indeksie
* Indeks jest na kolumnie, a zapytanie na wyrażeniu
* Wyszukanie zwraca zbyt wiele danych
* Query optimizer _myśli_, że zapytanie zwraca zbyt wiele danych

:::

```
SELECT * FROM users JOIN transactions
  ON transactions.user_id = users.id
  WHERE users.id = ?
    AND transactions.date > ?
```
## Ale to już inna historia...
