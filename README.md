## Tools
* Snowflake (Data Source)
* Snowflake (Data Warehouse)
* Airflow (Orchestration)
* DBT (Transformation)

## Step 1: Set up environment in snowflake
```
use role accountadmin;
create warehouse if not exists dbt_wh with warehouse_size='x-small';
create database if not exists dbt_db;
create role if not exists dbt_role;

show grants on warehouse dbt_wh;

grant usage on warehouse dbt_wh to role dbt_role;
grant role dbt_role to user "MYNAMEISRIYANDI";
grant all on database dbt_db to role dbt_role;


use role dbt_role;

create schema dbt_db.dbt_schema;
```
## Step 2: Configure dbt_project.yml and packages


## Step 3: Create source and staging tables


## Step 4: Transformed models (fact tables, data marts)


## Step 5: Create marcos Keep things D.R.Y (Don’t Repeat Yourself)


## Step 6: Generic and singular tests


## Step 7: Deploy models using Airflow


