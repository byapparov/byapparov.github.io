# Introduction

In this post I want to explore a data engineering framework that would
allow to achieve the following goals:
- Acceptance Test Driven Development (ATDD)
- Isolated code changes
- Shared documentation

There are few assumptions that proposed framework is designed for:
 - data pipelines that consist of a mix of SQL and code
 - there are multiple sources of data that are initially made available in a
  single datastore (data lake).
 - we are building a target data layer which will be available to the business

 This can be represented by this flow diagram:

```
 application
  -> data extraction     | service
  -> data lake           | datastore
  -> data transformation | batch
  -> data warehouse      | datastore
```

Our focus will be on the design of `data transformation` in a way that enables
teams to embrace Agile practices.

# Data Landscape

## Database

This framework is targeting any Data Warehousing tool and is technology agnostic. It has been implemented in BigQuery at MADE.com.

## Data Lake

We use term `data lake` to refer to all the data that comes from applications in the raw format. Usually this will be `json` files in cloud storage or data streamed as `json` into the database.  

We expect that data lake consists of tables without nested data.

If you have nested data, you will need to add pre-processing stage to flatten the data. We will refer to flat data layer as data lake going forward.

This separation will allow you to:
- simplify the queries that contain business logical;
- test your queries on the database engines that don't currently support nested data;
- reduce the size of the queries by additional separation layer;
- move data prep logic like duplicates removal, into pre-processing layer which will benefit all the queries in the data transformation layer.

# Work granularity

The most important element of the framework is based on the definition of the
unit of work.

We will require that each Story is targeting at least one and only one field
in the data warehouse.

This will allow stakeholders and developers to focus on requirements and
deliver the most important fields incrementally.

This will also allow developers to organise their code around tables and fields
in a way that changes can be integrated quickly.

# Capturing requirements

As many other product owners, we found spreadsheet to be the best way to
capture requirements for data.

Here is an example of requirements table and corresponding test data:

## Mock data for story acceptance

We can provide examples for the mock data that should capture a shared view between users, product owner and developers.

| datalake. customers.id | datalake. refunds.id | datalake. refunds.customer_id | datalake. refunds.amount |scenario|
|:---:|:---:|:---:|---:|:---|
|  1 |  1  |  1 | 50.00 | two refund records for customer 1
|  1 |  2  |  1 | 100.00|
|  2 |     |    |     | no refunds for customer 2
|  3 |  4   |  3  | 10.00 | duplicate refund|
|  3 |  4   |  3  | 10.00 |
|    |  5   | 100| 20.00 | orphan refund record


Now we can spec `attr_customers_all_refunds` staging output:

|customer_id|amount|comment|
|:---:|---:|:---|
|1|150.00|sum of the amount for two records|
|3|10.00|duplicate is removed|
|100|20.00|orphan record is included|

Requirements captured like this in spreadsheet can be done
using formulas which can be used as data lineage documentation.

These two tables can be easily converted into JSON files for mock `datalake.refunds` data for further ingestion into the database to test the query:

```JSON
{
  "id": 1,
  "customer_id": 1,
  "amount": 50.00
},
{
  "id": 2,
  "customer_id": 1,
  "amount": 100.00
},
{
  "id": 4,
  "customer_id": 3,
  "amount": 10.00
},
{
  "id": 4,
  "customer_id": 3,
  "amount": 10.00
},
{
  "id": 5,
  "customer_id": 100,
  "amount": 20.00
}
```

expected staging output in `attr_customers_all_refunds`:

```json
{
  "customer_id": 1,
  "amount": 150.00
},
{
  "customer_id": 3,
  "amount": 10.00
},
{
  "customer_id": 100,
  "amount": 20.00
}
```

Now developer can add SQL that conforms with the acceptance above, e.g.:

```sql
-- attr_customers_all_refunds
SELECT
  customer_id,
  SUM(amount) AS amount
FROM
(
  SELECT
    r.customer_id,
    r.amount,
    ROW_NUMBER() OVER(PARTITION BY id) rn
  FROM
    `datalake.refunds` r
)
WHERE
  -- this removes duplicate refund records
  rn = 1
GROUP BY
  -- getting data to the granularity of
  -- the relevant primary key
  customer_id
```

# Organising things

## Code

Simple and logical structure of the code will be driven by either individual
fields or data lake tables.

Let's explore an example for customers table.

We will build target `customers` table from `orders` and `refunds`.

```
.
└─customers
  │   pipeline.R
  └───pk
  │       fields.R
  └───orders
  │   │   first.sql
  │   │   first.R
  │   │   last.sql
  │   │   last.R
  │   └───tests
  │       │   test.R
  │       └───data
  │           └───first
  │           │       orders.json
  │           │       customers.json
  │           │       products.json
  │           └───last
  │                   orders.json
  │                   customers.json
  │                   products.json
  └───refunds
      │   all.sql
      │   all.ref
      └───tests
          │   test.R
          └───data
              └───all
                    refunds.json
                    customers.json
```

Each set of fields here will have its own set of test data which will have
minimum required set of fields and rows to cover examples from the Acceptance
documentation.

This organisation of the code base will allow you to:
- Map your folder structure to your target data
- Write independent SQL queries with unit tests


## Staging tables

For each set of fields we will create a staging table with the granularity of
the target table.

Here are the tables that will be created for the `customers` example:

* attr_customers_pk
* attr_customers_first_sale_order
* attr_customers_last_sale_order
* attr_customers_all_refunds

These tables in the first iteration can be combined with the simple query into the target table:

```sql
-- customers.sql
SELECT
  pk.id AS id,
  first_sale_order.date AS first_sale_order_date,
  last_sale_order.amount AS last_sale_order_amount,
  refunds.amount AS refund_amount
FROM
  attr_customers_pk pk
LEFT JOIN
  attr_customers_first_sale_order first_sale_order
  ON pk.id = first_sale_order.customer_id
LEFT JOIN
  attr_customers_last_sale_order last_sale_order
  ON pk.id = last_sale_order.customer_id
LEFT JOIN
  attr_customers_all_refunds refunds
  ON pk.id = refunds.customer_id
```

Assembly of the target table can be generalised through SQL builder
or process where each set of fields is added through a temporary table.

Image defining target table like this:

```yaml
# target.yaml
customers:
  - table   : attr_customers_pk:
    query   : customers/pk.sql
    key     : id
    fields:
      id: id
  - table   : attr_customers_first_sale_order:
    query   : customers/sale_order/first/fields.sql
    key     : customer_id
    fields:
      date: first_sale_order_date
  - table   : attr_customers_last_sale_order:
    query   : customers/sale_order/last/fields.sql
    key     : customer_id
    fields:
      amount: refunds_amount
  - table   : attr_customers_all_refunds:
    query   : customers/refunds/all/fields.sql
    key     : customer_id
    fields:
      amount: refund_amount
```

and with a line of code you build you target table:

```py
asseble_table("customers/target.yaml")
```

Orchestration, optimisation and validation are hidden in `assemble_table` function. Every time this function is improved, benefits are passed to all
the transformation pipelines that you maintain.

Note that location and names of the staging tables for individual attributes are not important as soon as engine can map them to the corresponding source query. For example, BigQuery stores results of every query execution in a temporary table for 24 hours. Transformation engine can leverage this feature and combine target table from temporary tables with random names. 

# Managing schema changes

## Adding fields

For each new field you will need to assess if you can add it to an existing
transformation query without changing granularity or subset of the data.

If that is the case you can extend existing acceptance case and update the
query accordingly.

Otherwise, you will need to build a new acceptance document and organise the new code as described above.

## Removing fields

We propose approach where fields are never hard removed from the target tables as
part of the story development.

Fields can be removed after a period of time once dependencies are cleared
from other projects that potentially rely on the the data point in question.

Mostly, field removal will be triggered in situation when quality of data
source is so low, that having no data is better than having `garbage` data.

In this case you can just default all the values in a given field to `(obsolete)`
 or other single value that works with your convention.

## Name changes

Change of the name should be executed as combination of addition of a field and
removal of a field.

For example, lets say you have an `amount` field in your table, and you want
to go for a more specific name `amount_ex_vat`.

It is possible that some services will have breaking dependency on `amount`
field.

Resolving all dependencies is impossible within a single release, unless you have a one code base for all projects. Instead, we will update metadata of `amount` field so it is obvious that `amount_ex_vat` should be used going forward. You can also set and communicate a target date for complete removal of `amount` field, at which point dependent projects can take responsibility for any outrage caused by removal of the field.


# Continuous integration

Code and test structure can be executed independently which means you can
run all the tests in parallel on small datasets which should result in
a very fast execution. Commonly used CI tools like Jenkins can help to achieve this.
