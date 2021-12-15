# DEMO STEPS :
## Preparing Dataset :
1. Create Dataset **demo_dbt**
2. Create tables : **customers** & **orders**
4. Fact table : run database -> req.sql

## Creating Credentials
  1. Create a service account **dbt-sa**
  2. Grant **dbt-sa** the role __*Bigquery Admin*__
  3. Generate json key for **dbt-sa**

## Creating a repository.
https://github.com/Abdelwaheb-Hnaien/dbt-tutorial

## Create and activate a virtual environment
```ruby
python3 -m venv dbt-env
source dbt-env/bin/activate
```
## Install DBT
https://docs.getdbt.com/dbt-cli/install/pip
```ruby
python3 -m pip install dbt-<adaptater> # dbt-bigquery
```
## Init dbt project
```ruby
dbt init dbt-tutorial
```

## Connect to BigQuery
When developing locally, dbt connects to your data warehouse using a profile — a yaml file with all the connection details to your warehouse.
1. Create a file in the ~/.dbt/ directory named profiles.yml.
2. Move your BigQuery keyfile into this directory.
3. Copy the following into the file — make sure you update the values where indicated.
```ruby
default:
    outputs:

      prod:
        type: bigquery
        method: oauth
        project: [GCP project id]
        dataset: [the name of your dbt dataset] # You can also use "schema" here
        threads: [1 or more]
        timeout_seconds: 300
        location: US # Optional, one of US or EU
        priority: interactive
        retries: 1

      dev:
        type: bigquery
        method: service-account
        project: test-ahn-dev
        dataset: Dataset_A
        threads: 1
        keyfile: "/Users/abdelwahebhnaien/.dbt/test-ahn-dev-creds.json"
        timeout_seconds: 300
        priority: interactive
        retries: 1

    target: dev
```
4. Execute the debug command from your project to confirm that you can successfully connect
```ruby
dbt debug
```
## Push local changes to remote repositories:
```ruby
git init
git remote add origin git@github.com:Abdelwaheb-Hnaien/dbt-tutorial.git
git branch -M main
git add .
git commit -m "initial commit"
git push -u origin main
```

## Create a model:

  1. Creat a file named **transactions.sql** under **models/**

  ```ruby
      with customers as (

        select
            customer_id,
            first_name,
            last_name

        from `test-ahn-dev`.base.customers

      ),

      orders as (

        select
            order_id,
            customer_id,
            order_date,
            status

        from `test-ahn-dev`.base.orders

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

      select * from final customer_id
  ```
  2. From the command line, execute `dbt run`

## Change the way your model is materialized
1. update dbt_project.yml and add `materialized: table` under dbt_tutorial.dbt_tutorial
```ruby
  models:
    dbt_tutorial:
      +materialized: table
      example:
        +materialized: view
```
2. Edit models/customers.sql to have the following snippet at the top:

  ```ruby
  {{
    config(
      materialized='view'
    )
  }}

  with customers as (

      select
          id as customer_id
          ...

  )
```

## Delete the example models
  1. Delete models/example
  2. Delete any configuration related to **example**
  ```ruby
    example:
      +materialized: view
  ```
  3. When you remove models from your dbt project, you should manually drop the related relations from your schema.

## Build models on top of other models
1. Create a new SQL file, **models/stg_customers.sql**, with the SQL from the customers CTE in our original query
```ruby
  select
      customer_id,
      first_name,
      last_name
  from `test-ahn-dev`.base.customers
```

2. Edit the SQL in your models/transactions.sql file as follows:
```ruby
  with customers as (

    select * from {{ ref('stg_customers')}}

  )
```
3. do the same for **models/stg_orders.sql**

## Writing bad SQL
 1. make a syntax error anywhere and then run **dbt run** -> you will notice a **Database Error**.
 2. Change the name of model reference and then then run **dbt run** -> you will notice a **Compilation Error**

## Test your model
  1. Create a new YAML file in the models directory, named models/schema.yml
  ```ruby
      version: 2

      models:
        - name: transactions
          description: One record per customer
          columns:
            - name: customer_id
              description: Primary key
              tests:
                - unique
                - not_null
            - name: first_order_date
              description: NULL when a customer has not yet placed an order.

        - name: stg_customers
          description: This model cleans up customer data
          columns:
            - name: customer_id
              description: Primary key
              tests:
                - unique
                - not_null

        - name: stg_orders
          description: This model cleans up order data
          columns:
            - name: order_id
              description: Primary key
              tests:
                - unique
                - not_null
            - name: status
              tests:
                - accepted_values:
                    values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']

  ```

  2. Execute **dbt test**, and confirm that all your tests passed.
  3. Chnage accepted values and restrict only t only one values.
  4. Run `dbt test`-> go to the target sql -> copy paste in BigQuery -> remove the last where condition to see all possible values -> head back to the test and update the list of accepted values.

## Document your model
  1. Create a file named model/transactions.md
  ```ruby
    {% docs transactions %}
    One record per customer

    [This is a link](google.com)

    * this is a list

    **bold text** _italic_

    |table|
    |-----|
    |description|
    {% enddocs %}

  ```

  2.Update **dbt_project.yml** and add a description for the model transactions :
  ```ruby
  models:
    - name: transactions
      description: "{{ doc('transactions') }}"
      ...
  ```

### Resources:
- Learn more about dbt [in the docs](https://docs.getdbt.com/docs/introduction)
- Check out [Discourse](https://discourse.getdbt.com/) for commonly asked questions and answers
- Join the [chat](https://community.getdbt.com/) on Slack for live discussions and support
- Find [dbt events](https://events.getdbt.com) near you
- Check out [the blog](https://blog.getdbt.com/) for the latest news on dbt's development and best practices
- Bigquery config [here](https://docs.getdbt.com/reference/resource-configs/bigquery-configs)
