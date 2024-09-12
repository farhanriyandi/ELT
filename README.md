## Tools
* Snowflake (Data Source)
* Snowflake (Data Warehouse)
* Airflow (Orchestration)
* DBT (Transformation)

## Step 1: Set up environment in snowflake
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
## Step 2: Configure dbt_project.yml and packages


## Step 3: Create source and staging tables


## Step 4: Transformed models (fact tables, data marts)


## Step 5: Create marcos Keep things D.R.Y (Don’t Repeat Yourself)


## Step 6: Generic and singular tests


## Step 7: Deploy models using Airflow


