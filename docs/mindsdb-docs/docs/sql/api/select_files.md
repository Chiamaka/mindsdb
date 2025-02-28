# `#!sql SELECT * FROM files.[file_name]` Statement

## Description

The `#!sql SELECT * FROM files.[file_name]` statement is used to select data from a file.

First, you upload a file to the MindsDB Cloud Editor by following [this guide](/sql/create/file/). And then, you can [`CREATE PREDICTOR`](/sql/create/predictor/) from the uploaded file.

## Syntax

Here is the syntax:

```sql
SELECT *
FROM files.[file_name];
```

On execution, we get:

```sql
+--------+--------+--------+--------+
| column | column | column | column |
+--------+--------+--------+--------+
| value  | value  | value  | value  |
+--------+--------+--------+--------+
```

Where:

| Name          | Description                                                                                           |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| `[file_name]` | Name of the file uploaded to the MindsDB Cloud Editor by following [this guide](/sql/create/file/).   |
| `column`      | Name of the column from the file.                                                                     |

## Example

Once you uploaded your file by following [this guide](/sql/create/file/), you can query it like a table.

```sql
SELECT *
FROM files.home_rentals
LIMIT 10;
```

On execution, we get:

```sql
+-----------------+---------------------+-------+----------+----------------+---------------+--------------+--------------+
| number_of_rooms | number_of_bathrooms | sqft  | location | days_on_market | initial_price | neighborhood | rental_price |
+-----------------+---------------------+-------+----------+----------------+---------------+--------------+--------------+
| 0               | 1                   | 484,8 | great    | 10             | 2271          | south_side   | 2271         |
| 1               | 1                   | 674   | good     | 1              | 2167          | downtown     | 2167         |
| 1               | 1                   | 554   | poor     | 19             | 1883          | westbrae     | 1883         |
| 0               | 1                   | 529   | great    | 3              | 2431          | south_side   | 2431         |
| 3               | 2                   | 1219  | great    | 3              | 5510          | south_side   | 5510         |
| 1               | 1                   | 398   | great    | 11             | 2272          | south_side   | 2272         |
| 3               | 2                   | 1190  | poor     | 58             | 4463          | westbrae     | 4123.812     |
| 1               | 1                   | 730   | good     | 0              | 2224          | downtown     | 2224         |
| 0               | 1                   | 298   | great    | 9              | 2104          | south_side   | 2104         |
| 2               | 1                   | 878   | great    | 8              | 3861          | south_side   | 3861         |
+-----------------+---------------------+-------+----------+----------------+---------------+--------------+--------------+
```

Now let's create a predictor using the uploaded file. You can learn more about the [`CREATE PREDICTOR` command here](/sql/create/predictor/).

```sql
CREATE PREDICTOR mindsdb.home_rentals_model
FROM files
    (SELECT * from home_rentals)
PREDICT rental_price;
```

On execution, we get:

```sql
Query OK, 0 rows affected (x.xxx sec)
```
