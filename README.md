# avro-tools

sqoop import-all-tables -Dmapreduce.job.user.classpath.first=true \
  --connect "jdbc:mysql://nn01.itversity.com:3306/retail_db" \
  --username=retail_dba \
  --password=itversity \
  -m 1 \
  --as-avrodatafile \
  --warehouse-dir /user/gnanaprakasam/sqoop_import_avro

sqoop import -Dmapreduce.job.user.classpath.first=true \
  -m 1 \
  --connect "jdbc:mysql://nn01.itversity.com:3306/retail_db" \
  --username=retail_dba \
  --table orders \
  --password=itversity \
  --as-avrodatafile \
  --warehouse-dir=/user/gnanaprakasam/sqoop_import_avro


hadoop fs -get /user/gnanaprakasam/sqoop_import_avro/departments .

avro-tools getschema part-m-00000.avro > department.avsc

hadoop fs -put /home/gnanaprakasam/departments/department.avsc /user/gnanaprakasam/sqoop_import_avro

create external table departments
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/departments'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/department.avsc');


hadoop fs -get /user/gnanaprakasam/sqoop_import_avro/categories .

avro-tools getschema part-m-00000.avro > category.avsc

hadoop fs -put /home/gnanaprakasam/categories/category.avsc /user/gnanaprakasam/sqoop_import_avro

create external table categories
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/categories'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/category.avsc');

avro-tools tojson part-m-00000.avro > category.json

avro-tools fromjson category.json --schema-file category.avsc


# Add columns in create table and remove it from avsc files.

hadoop fs -put /home/gnanaprakasam/categories/category_bkup.avsc /user/gnanaprakasam/sqoop_import_avro

create external table categories(
category_id int,
category_department_id int,
category_name string)
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/categories'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/category_bkup.avsc');

create external table categories(
category_id int,
category_department_id int,
category_name string)
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/categories'
tblproperties('avro.schema.literal' = '
{
  "type" : "record",
  "name" : "categories",
  "doc" : "Sqoop import of categories",
  "fields" : [ {
    "name" : "category_id",
    "type" : [ "null", "int" ],
    "default" : null
  }, {
    "name" : "category_department_id",
    "type" : [ "null", "int" ],
    "default" : null
  }, {
    "name" : "category_name",
    "type" : [ "null", "string" ],
    "default" : null
  } ],
  "tableName" : "categories"
}');

# Use paritioned in create table

hadoop fs -get /user/gnanaprakasam/sqoop_import_avro/orders .

avro-tools getschema part-m-00000.avro > orders.avsc

hadoop fs -put /home/gnanaprakasam/orders/orders.avsc /user/gnanaprakasam/sqoop_import_avro

create table orders_part_avro(
order_id int,
order_date string,
order_customer_id int,
order_status string)
partitioned by (order_month string)
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/orders_part_avro'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/orders_part_avro.avsc');

hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/orders_part_avro

set hive.exec.dynamic.partition;
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode;
set hive.exec.dynamic.partition.mode=nonstrict;

insert into table orders_part_avro partition (order_month)
select order_id, order_date, order_customer_id, order_status, substr(order_date, 1, 7) order_month from orders;


select order_status, count(1) from orders where substr(order_date, 1, 7) like '2014-01%'  group by order_status;
Total MapReduce CPU Time Spent: 28 seconds 90 mse
Time taken: 24.374 seconds, Fetched: 9 row(s)

select order_status, count(1) from orders_part_avro where substr(order_date, 1, 7) like '2014-01%' group by order_status;
Total MapReduce CPU Time Spent: 9 seconds 550 msec
Time taken: 18.896 seconds, Fetched: 9 row(s)

--------------------------------------------------------------------------------------------------------------------------------
create table orders
stored as avro
location '/user/gnanaprakasam/sqoop_import_avro/orders'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/orders.avsc');

select * from orders where from_unixtime(cast(substr(order_date, 1, 10) as int)) like '2014-01%';


create table orders_part_avro(
order_id int,
order_date bigint,
order_customer_id int,
order_status string)
partitioned by (orders_month string)
stored as avro
location 'hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/orders_part_avro'
tblproperties('avro.schema.url' = '/user/gnanaprakasam/sqoop_import_avro/orders_part_avro.avsc');

dfs -ls hdfs://nn01.itversity.com:8020/user/gnanaprakasam/sqoop_import_avro/orders_part_avro

alter table orders_part_avro add partition (orders_month = '2014-01');

insert into table orders_part_avro partition (orders_month = '2014-01')
select * from orders where from_unixtime(cast(substr(order_date, 1, 10) as int)) like '2014-01%';

drop table orders_part_avro;

set hive.exec.dynamic.partition;
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode;
set hive.exec.dynamic.partition.mode=nonstrict;

# Create the orders_part_avro table again

insert into table orders_part_avro partition (orders_month)
select order_id, order_date, order_customer_id, order_status, substr(from_unixtime(cast(substr(order_date, 1, 10) as int)), 1, 7) orders_month from orders;

--------------------------------------------------------------------------------------------------------------------------------

# Add two columns in orders

hadoop fs -put -f /home/gnanaprakasam/departments/department.avsc /user/gnanaprakasam/sqoop_import_avro

select * from departments;

insert table departments values (500,500,Testing,null);


