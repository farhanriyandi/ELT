I learned about ELT on the YouTube channel `jayzern` [Click this link for the YouTube channel that I followed to learn on this course](https://www.youtube.com/watch?v=OLXkGB7krGo)

## Tools
* Snowflake (Data Source)
* Snowflake (Data Warehouse)
* Airflow (Orchestration)
* DBT (Transformation)

## Step 1 Set up environment in snowflake

```
use role accountadmin; -- This sets the role used to ACCOUNTADMIN, which is the role with the highest level of access in Snowflake. This role allows the execution of all types of administrative operations
create warehouse if not exists dbt_wh with warehouse_size='x-small'; -- Create a warehouse named dbt_wh with an x-small size if it does not already exist
create database if not exists dbt_db; -- Create a database named dbt_db if it does not already exist.
create role if not exists dbt_role; -- Create a role named dbt_role if it does not already exist. The role is used to manage user access permissions in Snowflake

show grants on warehouse dbt_wh; -- This helps verify who has access to the dbt_wh warehouse.

grant usage on warehouse dbt_wh to role dbt_role; -- Grant usage rights for the dbt_wh warehouse to the dbt_role role
grant role dbt_role to user "MYNAMEISRIYANDI"; -- Grant the dbt_role role to the user named 'MYNAMEISRIYANDI'.
grant all on database dbt_db to role dbt_role; -- Grant all permissions on the dbt_db database (such as SELECT, INSERT, UPDATE, DELETE) to the dbt_role role.


use role dbt_role; -- Change the current role to dbt_role, so that subsequent commands are executed with the permissions granted to this role

create schema dbt_db.dbt_schema; -- Create a schema named dbt_schema within the dbt_db database. The schema is used to organize tables and other objects within the database.
```

## Step 2: Setup DBT Project

### Create and activate a virtual environment
```
python3 -m venv dbt-venv
source venv/bin/activate
```
   
### Install and Setup dbt
Install dbt-snowflake
```
pip install dbt-snowflake
```

And also install dbt-core
```
pip install dbt-core
```

### To create a dbt project, use dbt init. Here, we name our project data_pipeline. Run the following command:
```
dbt init data_pipeline
```

### After that you will get some questions like the following picture:
![image](https://github.com/user-attachments/assets/f9a647ea-9ce7-4972-9b62-89b8213bbdb9)


![image](https://github.com/user-attachments/assets/545a92c1-52e4-46ae-94c7-1a9bda997e19)


### Change to the data_pipeline directory.
```
cd data_pipeline
```

## Step 3: Setup DBT Profile
By default, DBT will create a dbt profile at your home directory `~/.dbt/profiles.yml`
You can update the profiles, or you can make a new dbt-profile directory.
To make a new dbt-profie directory, you can invoke the following:
```
mkdir dbt-profiles
touch dbt-profiles/profiles.yml
export DBT_PROFILES_DIR=$(pwd)/dbt-profiles
```

Set profiles.yml as follow:
```
data_pipeline:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: aw18386.us-central1.gcp # use your account locator
      user: MYNAMEISRIYANDI # change it to your username
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}" 
      role: dbt_role
      database: dbt_db
      warehouse: dbt_wh
      schema: dbt_schema
      threads: 1
      client_session_keep_alive: False
```


![image](https://github.com/user-attachments/assets/d7ebffe4-f403-4690-9de7-5b6e20232672)
For the account locator, you can go to Admin, then Accounts. In the locator section, you can copy it. In my account locator, the result is: `https://aw18386.us-central1.gcp.snowflakecomputing.com`
You can place it in the `profiles.yml` under the locator section like this: `aw18386.us-central1.gcp`

And in the terminal, you can type the following:
```
export SNOWFLAKE_PASSWORD=your_actual_password
```
Then, in the `profiles.yml`, you can enter your password as shown in the profiles.yml above like this one `password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"`



## Step 4: Configure dbt_project.yml and packages
### Setup DBT Project configuration in dbt_project.yml
```
models:
  data_pipeline:
    # Config indicated by + and applies to all files under models/example/
    staging:
      +materialized: view
      snowflake_warehouse: dbt_wh
    marts:
      +materialized: table
      snowflake_warehouse: dbt_wh
```

### Next, install dbt-utils, which will be useful for performing generic tests. 
1. Add a packages.yml file to your project.

2. In the `packages.yml`, add the following code: 
```
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
```
3. Run `dbt deps` in your terminal to download and install the dependencies defined in the packages.yml file for your dbt project.


## Step 5: Create source and staging tables
In the `models` folder, add a folder named `staging`.
In the `staging` folder, add files named `tpch_source.yml`, `stg_tpch_orders.sql`, and `stg_tpch_line_items.sql`.

Modification your `tpch_source.yml`
```
version: 2

sources:
  - name: tpch
    database: snowflake_sample_data
    schema: tpch_sf1
    tables:
      - name: orders
        columns:
          - name: o_orderkey
            data_tests:
              - unique
              - not_null
      - name: lineitem
        columns:
          - name: l_orderkey
            data_tests:
              - relationships:
                  to: source('tpch', 'orders')
                  field: o_orderkey

```

Modification your `stg_tpch_orders.sql`
```
select 
    o_orderkey as order_key,
    o_custkey as customer_key,
    o_orderstatus as status_code,
    o_totalprice as total_price,
    o_orderdate as order_date
from {{ source('tpch', 'orders') }}
```

Modification your `stg_tpch_line_items.sql`
```
select 
    {{
        dbt_utils.generate_surrogate_key([
            'l_orderkey',
            'l_linenumber'
        ])
    }} as order_item_key,
    l_orderkey as order_key,
    l_partkey as part_key,
    l_linenumber as line_number,
    l_quantity as quantity,
    l_extendedprice as extended_price,
    l_discount as discount_percentage,
    l_tax as tax_rate
from 
    {{ source('tpch', 'lineitem') }}
```


## Step 6: Transformed models (fact tables, data marts)

In the `models` folder, add a folder named `marts`. In the `marts` folder, add files named `int_order_items.sql`, `int_order_items_summary.sql`, and 'fct_orders.sql'

Modification your `int_order_items.sql`
```
select 
    line_item.order_item_key,
    line_item.part_key,
    line_item.line_number,
    line_item.extended_price,
    orders.order_key,
    orders.customer_key,
    orders.order_date,
    {{ discounted_amount('line_item.extended_price', 'line_item.discount_percentage')}} as item_discount_amount
from 
    {{ ref('stg_tpch_orders') }} as orders 
join
    {{ ref('stg_tpch_line_items') }} as line_item  
        on orders.order_key = line_item.order_key
order by
    orders.order_date
```

Modification your `int_order_items_summary.sql`
```
select
    order_key,
    sum(extended_price) as gross_item_sales_amount,
    sum(item_discount_amount) as item_discount_amount
from
    {{ ref('int_order_items') }}
group by
    order_key
```

Modification your `fct_orders.sql`
```
select
    orders.*,
    order_item_summary.gross_item_sales_amount,
    order_item_summary.item_discount_amount
from
    {{ ref('stg_tpch_orders') }} as orders 
join
    {{ ref('int_order_items_summary') }} as order_item_summary 
        on orders.order_key = order_item_summary.order_key
order by order_date
```

## Step 7: Create marcos Keep things D.R.Y (Don’t Repeat Yourself)

**Macros** are used to create reusable SQL code. By using macros, you can write SQL logic once and then use it in multiple models or other places within your dbt project. This helps reduce code duplication and makes the project more modular and easier to manage.

In the `macros` folder, add a file named `pricing.sql`.
```
{% macro discounted_amount(extended_price, discount_percentage, scale=2) %}
    (-1 * {{ extended_price }} * {{discount_percentage}})::decimal(16, {{ scale }})
{% endmacro %}
```

## Step 8: Generic and singular tests+

### For Generic Tests
In the models/marts folder, add a file named `generic_tests.yml`.
```
models:
  - name: fct_orders
    columns:
      - name: order_key
        tests:
          - unique
          - not_null
          - relationships:
              to: ref('stg_tpch_orders')
              field: order_key
              severity: warn
      - name: status_code
        tests:
          - accepted_values:
              values: ['P', 'O', 'F']
```

### For singular tests
In the `tests` folder, add a file named `fct_orders_discount.sql`.
```
select
    *
from
    {{ref('fct_orders')}}
where
    item_discount_amount > 0
```

In the `tests` folder, add a file named `fct_orders_date_valid.sql`.
```
select 
    *
from
    {{ref('fct_orders')}}
where
    date(order_date) > CURRENT_DATE()
    or date(order_date) < date('1990-01-01')
```

### Run and test the models
```
dbt run && dbt test
```

## Step 9: Deploy models using Airflow
The next step is to move back one directory from the `data_pipeline directory`, then create a new folder which we will name `dbt_dag`

```
cd ..
mkdir dbt-dag
cd dbt-dag
```

And then run
```
astro dev init
```

Before running Airflow, by default airflow will run on `localhost:8080`, but we can set the port as desired. In the `.astro` folder, there is a file called `config.yml`. Here, I set the PostgreSQL port to `5435` and the Airflow port to `8089`. So i have to use `localhost:8089` to open airflow

```
project:
  name: dbt-dag
webserver:
  port: 8089
postgres:
  port: 5435
```


After that run
```
astro dev start
```

In dbg `dags` folder there is `dbt_dag.py` add this code
```
import os
from datetime import datetime

from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import SnowflakeUserPasswordProfileMapping


profile_config = ProfileConfig(
    profile_name="default",
    target_name="dev",
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id="snowflake_conn", 
        profile_args={"database": "dbt_db", "schema": "dbt_schema"},
    )
)

dbt_snowflake_dag = DbtDag(
    project_config=ProjectConfig("/usr/local/airflow/dags/dbt/data_pipeline",),
    operator_args={"install_deps": True},
    profile_config=profile_config,
    execution_config=ExecutionConfig(dbt_executable_path=f"{os.environ['AIRFLOW_HOME']}/dbt_venv/bin/dbt",),
    schedule_interval="@daily",
    start_date=datetime(2024, 9, 9),
    catchup=False,
    dag_id="dbt_dag",
)
```

The data_pipeline folder is then copied and pasted into the dags/dbt folder as follows:
![image](https://github.com/user-attachments/assets/bc7582f8-4b1b-4688-9757-46c9bdfacf42)


And then open `localhost:8089` to access airflow. The username is `admin` and the password is `admin`. After that, click on Admin and then click on connections to create a connection between Airflow and Snowflake.

![image](https://github.com/user-attachments/assets/a5787e19-0cb8-49ff-8b0a-e6e28f88d2c0)

![image](https://github.com/user-attachments/assets/b4f18c20-b3d0-4318-91aa-4e86a94fb058)

### Trigger DAG
![image](https://github.com/user-attachments/assets/479344fb-cca4-49cf-9ff1-366b438d38d0)

![image](https://github.com/user-attachments/assets/2fe0f66a-d93a-4574-98fa-e88c19b5ae26)







