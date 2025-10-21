![alt text](image.png)

![alt text](image-1.png)
![alt text](image-2.png)
![alt text](image-3.png)
![alt text](image-4.png)
![alt text](image-5.png)
![alt text](image-6.png)
![alt text](image-8.png)
![alt text](image-9.png)
![alt text](image-10.png)
![alt text](image-11.png)
![alt text](image-12.png)

# Cloning Tables

```SQL
// Cloning

SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS;

CREATE TABLE OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE
CLONE OUR_FIRST_DB.PUBLIC.CUSTOMERS;


// Validate the data
SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;


// Update cloned table

UPDATE OUR_FIRST_DB.public.CUSTOMERS_CLONE
SET LAST_NAME = NULL;

SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS ;

SELECT * FROM OUR_FIRST_DB.PUBLIC.CUSTOMERS_CLONE;






// Cloning a temporary table is not possible
CREATE OR REPLACE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TEMP_TABLE(
  id int);

CREATE TRANSIENT TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;
```

![alt text](image-13.png)
![alt text](image-14.png)
```SQL
CREATE TEMPORARY TABLE OUR_FIRST_DB.PUBLIC.TABLE_COPY
CLONE OUR_FIRST_DB.PUBLIC.TEMP_TABLE;
```
![alt text](image-15.png)

```SQL
SELECT * FROM OUR_FIRST_DB.PUBLIC.TABLE_COPY;
```