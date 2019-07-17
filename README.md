# Postgres

## In the terminal

```zsh
psql --help # ask for help
paql -l # list all database
psql -d [DATABASE_NAME] # connect to the the database (can ignore -d)
```

## Postgres CLI

```zsh
\l # list all database
\d # list all relations
\dt # list all tables
\d [TABLE_NAME] # describe table
\df # list all available functions
\c [DATABASE_NAME] # switch between databases
\i [PATH_OF_THE_FILE] # execute commands from file
\x # expand display
\? # help
\q # exit
```

### copy a table from remote server

`pg_dump -U <username> -h <host> -t <table> <database> | psql <database>`

```zsh
pg_dump -U mlisa -h 35.224.208.150 -t metric mlisa | psql mlisa
```

### Create and drop database or table

> Primary keys uniquely identify a record in tables.

```sql
# database
CREATE DATABASE hello;
DROP DATABASE hello;

# table
CREATE TABLE person (
  id bigserial NOT NULL PRIMARY KEY,
  first_name varchar(50) NOT NULL,
  last_name varchar(50) NOT NULL,
  date_of_birth date NOT NULL,
  email varchar(100)
);
DROP TABLE person;
```

### Insert records

```sql
INSERT INTO person (first_name, last_name, date_of_birth) VALUES ('kevin', 'hsiao', date '1987-03-25');
```

### Delete records

```sql
DELETE FROM person WHERE id=10;
```

### Update records

```sql
UPDATE person SET email='kevin@ruckus.com', age=32 WHERE id=100;
```

#### ON CONFLICT

```sql
# ON CONFLICT DO NOTHING
# will not work if you do not have the UNIQUE column(have constraints, either PRIMARY KEY or UNIQUE CONSTRAINT)
INSERT INTO person (id, first_name, last_name, date_of_birth, email)
VALUES (1, 'kevin', 'hsiao', date '1987-03-25', 'kevin@ruckus.com')
ON CONFLICT (id) DO NOTHING;

# ON CONFLICT DO SOMETHING
INSERT INTO person (id, first_name, last_name, date_of_birth, email, timestamp)
VALUES (1, 'kevin', 'hsiao', date '1987-03-25', 'kevin@ruckus.com', 1557657749057)
ON CONFLICT (id) DO UPDATE email=EXCLUDED.email, timestamp=EXCLUDED.timestamp;
```

### Read records

#### SELECT...FROM

```sql
# basic
SELECT * FROM person;
SELECT first_name FROM person;
SELECT first_name, last_name FROM person;

# arithmetic operators
SELECT id, make, model, price, ROUND(price * .90, 2) FROM car;

# alias(to override the original column name)
SELECT id, make, model, price AS original_price, ROUND(price * .90, 2) AS price_after_10_percent_discount FROM car;
```

#### ORDER BY: default is ASC

```sql
SELECT * FROM person ORDER BY country_of_birth ASC;
SELECT * FROM person ORDER BY country_of_birth DESC;
```

#### DISTINCT

```sql
SELECT DISTINCT country_of_birth FROM person ORDER BY country_of_birth ASC;
```

#### WHERE

```sql
# WHERE (with AND/OR)
SELECT * FROM person WHERE gender='Male' AND (nationality='Taiwan' OR nationality='Singapore');

# Comparison operators
# =, >=, <=, <>

# IN
SELECT * FROM person WHERE country_of_birth IN ('Taiwan', 'Singapore');
# is equal to
SELECT * FROM person WHERE country_of_birth='Taiwan' OR country_of_birth='Singapore';

# BETWEEN
SELECT * FROM person WHERE date_of_birth BETWEEN DATE '2000-01-01' AND DATE '2010-01-01' ORDER BY date_of_birth ASC;

# LIKE AND ILIKE(case insensitive)
SELECT * FROM person WHERE email LIKE '%@google.%';
SELECT * FROM person WHERE country_of_birth LIKE 'p%';
```

#### LIMIT and OFFSET

```sql
# Limit and offset
SELECT * FROM person LIMIT 10
SELECT * FROM person OFFSET 5 LIMIT 5
```

#### Aggregate functions: MAX / MIN / AVG / SUM

```sql
# Aggregate functions: max/min/avg/sum
SELECT MIN(price) from car;
SELECT MAX(price) from car;
SELECT ROUND(AVG(price)) from car;
```

#### :astonished: GROUP BY

`GROUP BY`: single or multiple columns

`GROUP BY ...HAVING(condition)`: `WHERE` clause is used to place conditions on columns but what if we want to place conditions on groups? We can use `HAVING` clause to place conditions to decide which group will be the part of final result-set. Also we can not use the aggregate functions like `SUM()`, `COUNT()` etc.

```sql
# single column
SELECT country_of_birth, COUNT(*) FROM person GROUP BY country_of_birth ORDER BY country_of_birth;
# multiple columns
SELECT car_make, car_model, ROUND(AVG(price)) FROM car GROUP BY car_make, car_model ORDER BY car_make;

# with HAVING clause
SELECT country_of_birth, COUNT(*) FROM person GROUP BY country_of_birth HAVING COUNT(*) >= 5 ORDER BY country_of_birth;
```

#### COALESCE: to give default value

```sql
SELECT COALESCE(email, 'Email not provided') FROM person;

# COALESCE with NULLIF (to handle division by 0)
SELECT COALESCE(numerator/NULLIF(denominator, 0), 0) from question;

SELECT 0/0; # because throws ERROR
SELECT 0/NULL; # but no ERROR, just empty value
```

#### DATE

```sql
# Date/Time
SELECT NOW(); # 2019-05-12 16:55:22.445841+08
SELECT NOW()::DATE; # 2019-05-12
SELECT NOW()::TIME; # 16:55:22.445841

# Adding and subtracting with Dates
SELECT NOW() - INTERVAL '3 YEARS';
SELECT NOW() + INTERVAL '1 DAY';
SELECT (NOW() + INTERVAL '5 MONTHS')::DATE;

# Extracting fields
SELECT EXTRACT(MONTH FROM NOW());
SELECT EXTRACT(DOW FROM NOW()); # Date of weak
SELECT EXTRACT(CENTURY FROM NOW());

# Age function
SELECT AGE('2019-01-01', '1987-01-01'); # 32
SELECT first_name, last_name, email, gender, date_of_birth, EXTRACT(YEAR FROM AGE(NOW(), date_of_birth)) AS age FROM person;
```

#### CONSTRAINT: UNIQUE and CHECK

```sql
# remove constraint
ALTER TABLE person DROP CONSTRAINT person_pkey;
# add constraint
ALTER TABLE person ADD CONSTRAINT person_pkey(id);

# UNIQUE constraints
ALTER TABLE person ADD CONSTRAINT unique_email_address UNIQUE (email);
ALTER TABLE person ADD UNIQUE (email); # the constraints name will be defined by postgres

# customized constraints
ALTER TABLE person ADD CONSTRAINT gender_constraint CHECK (gender='Female' OR gender='Male');
ALTER TABLE person ADD CONSTRAINT gender_constraint CHECK (gender IN ('Female', 'Male'));
```

### Foreign keys, Joins and Relationships

```sql
# We have two tables
CREATE TABLE person (
  id BIGSERIAL NOT NULL PRIMARY KEY,
  # ...
  car_id BIGINT REFERENCES car (id),
  UNIQUE(car_id)
);

CREATE TABLE car (
  id BIGSERIAL NOT NULL PRIMARY KEY,
  # ...
);
```

#### INNER JOIN and LEFT JOIN

> We are not able to delete a record that has been referenced by other places. We need to remove that reference first.

```sql
# INNER JOIN returns those with foreign key
SELECT person.first_name, person.last_name, car.make, car.model, car.price FROM person JOIN car ON person.car_id=car.id;

# USING: if both the foreign key and the primary key are the same name
SELECT person.first_name, person.last_name, car.make, car.model, car.price FROM person JOIN car USING (car_id);


# LEFT JOIN returns those with or without foreign key
SELECT person.first_name, person.last_name, car.make, car.model, car.price FROM person LEFT JOIN car ON person.car_id=car.id;
```

### Export query results to csv file

```sql
\copy (SELECT * FROM person LEFT JOIN car ON car.id=person.car_id) TO '/Users/kevin/Downloads/result.csv' DELIMITER ',' CSV HEADER;
```

### Serial & Sequences

```sql
# will increase the id
SELECT nextval('person_id_seq'::regclass);
# reset the id sequence with 10
ALTER SEQUENCE person_id_seq RESTART WITH 10;
```

### Extensions

```sql
# see all the available extensions
SELECT * FROM pg_available_extensions;
# install extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
# create a uuid
SELECT uuid_generate_v4();

```

References:

- [Data types](https://www.postgresql.org/docs/current/datatype.html)
- [Date/Time types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- [Aggregate functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
