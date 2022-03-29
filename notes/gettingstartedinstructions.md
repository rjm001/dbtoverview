## Getting Started

1. Setup big query with this query

```sql
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from `dbt-tutorial`.jaffle_shop.customers

),

orders as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from `dbt-tutorial`.jaffle_shop.orders

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),


final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final

```

- Note, to load your own data into BigQuery, you would want to use an off-the-shelf tool like `Stitch` or `Fivetran`. `dbt` cannot be used to extract or load data. It only does transformations.


2. Generate BigQuery Credentials
    - You need a keyfile
    - this is analogous to using a database user name and password with most other data warehouses

- Use the following credential options of the BigQuery credential wizard
- API: BigQuery API
- data? Application data (creating a service account)
    - so don't need OAuth client
- Not planning to use this API with App Engine or Compute Engine
- Service account name: dbt-user
- Role: BigQuery Job User, BigQuery User, and BigQuery Data Editor
- Key type: JSON
- download the json file and save it in an easy-to-remember spot with a clear filename (dbt-user-crds.json) but **not in your github repo**

3. generate your dbt project
    a. `dbt --version` #check it's working
    b. `dbt init jaffle_shop`
    c. `cd jaffle_shop`

4. connect to BigQuery
    a. navigate to the `~/.dbt` directory
    b. put the BigQuery keyfile into this directory (get it from BigQuery website)
    c. create a file named `profiles.yml` in the same directory and copy the following text into it, then perform the requested changes.

```yml
jaffle_shop: # this needs to match the profile: in your dbt_project.yml file
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      keyfile: /Users/claire/.dbt/dbt-user-creds.json # replace this with the *full path* to your keyfile. Note, you cannot abbreviate the full path with ~/.dbt/dbt-user-creds.json. It will fail!
      project: grand-highway-265418 # Replace this with your project id. from bigquery. Should be in quotes? it isn't here.
      dataset: dbt_alice # Replace this with dbt_your_name, e.g. dbt_bob
      threads: 1
      timeout_seconds: 300
      location: US
      priority: interactive
t
```
    d. navigate to `jaffle_shop` directory that was created when you executed `dbt init` and execute `dbt debug`. If you have successfully performed all the above steps, it should say "Connection test: OK connection ok"

5. Perform your first dbt run
    a. Execute `dbt run` from `jaffle_shop`


### Build your first models

1. create a file `jaffle_shop/models/customers.sql`
2. paste the SQL query from the start of the document in that file
3. execute `dbt run`
4. note that 
    - `target/compiled` is a directory of compiled select statements
    - `target/run/` directory is compiled `create` statements
    - the `logs/dbt.log` file has verbose logging

### Change the way your model is materialized

1. Use the + sign in your dbt_project.yml

```
models:
  jaffle_shop:
    +materialized: table
    example:
      +materialized: view
```
- the above makes it so customers is a table

2. Add the following to `models/customers.sql` file
    - This makes it so the materialization of customers is as a view, rather than a table
```sql
{{
  config(
    materialized='view'
  )
}}

```
- execute dbt run

- run `dbt run --full-refresh` to make sure the changes updated on BigQuery
- note you now have `customers` as a view on BigQuery

3. delete the example models; i.e., delete all of `models/example`, including directory

## Build Models on top of other Models

create a new file `models/stg_customers.sql` with the following content:

```sql
select
    id as customer_id,
    first_name,
    last_name

from `dbt-tutorial`.jaffle_shop.customers

```

create another file as `models/stg_orders.sql` with the following content:

```sql
select
    id as order_id,
    user_id as customer_id,
    order_date,
    status

from `dbt-tutorial`.jaffle_shop.orders
```

4. Edit models/customsers.sql