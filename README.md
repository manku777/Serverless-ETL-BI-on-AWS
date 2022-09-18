# Serverless-ETL-BI-on-AWS

## PROBLEM STATEMENT

![image](https://user-images.githubusercontent.com/87073324/190860423-1bc878d8-9ced-4290-8aac-bf50437d85a3.png)



1. END POINTS are used to connect to RDS instance remotely, so make a note of it. So go to my SQL workbench or anything and click in "New Database Connection". End point shoudl be the host name while connecting, port will be 3306, also provide username and password. 


## How to load data from LOCAL TO RDS

First create the table in local and use the below command to insert the data from local to RDS instance. RDS and MYSQL local are connected through the end point which we created in the AWS RDS cluster

LOAD DATA LOCAL INFILE '/your_local_system_path/table_name.csv' 
INTO TABLE table_name FIELDS TERMINATED BY ','
ENCLOSED BY '"' IGNORE 1 LINES;

## Create a Redshift cluster

- Created IAM role, cluster configuration, select 2 nodes, make sure toh select YES at publicly accessible option so that we can connect to redshift from local SQL    client. In order to allow local SQL client to access Redshift, make sure you add Redshift in the VPC and select port 5439 and source will be my IP
- Once this is setup, we can use the endpoint and add it to the local SQL client(DBeaver), we also need to provide Database, user, password and test connection
- We have created two table - mysql_dwh and my_dwh_staging

## S3 bucket
- We will create folder for all the table in S3 bucket and under the folder we'll have "historical" named folder where we are going to store the particular table data. So, 10 tables -> 10 folders- 10 historical subfolders. This we need to configure while creating the data pipeline(Output S3 folder)
 * Once we have our historical data csv dumped in the S3, we'll create a schema(mysql_dwh) in Redshift. Under that schema create table orders(which was create by joining customer and orders table to avoid joining). Order of the columns in SQL query and order of columns in table should be same
 * Syntax for dumping data from S3 to Redshift
 
   copy schema.tablename from 's3://mysql-dwh/orders/historical/orders.csv'                  
   iam_role: 'arn:aws:iam::432432432:role/sdfsdf'                           // can get from the iam role which we created for Redshift
   CSV QUOTE '\"' DELIMITER ','
   acceptinvchars;                                                         // if any special chars, convert it into character
   
## AWS Glue
- AWS Glue is a fully managed ETL that makes it easy for customers to prepare and load their data in Redhshift for analytics. We simply point AWS Glue to our data stored on AWS, AWS Glue discovers our data and stores the associated metadata(table definition and schema) in the AWS Glue Data Catalog.
- AWS Glue has two main components
  * Glue Jobs
  * Glue Crawlers
- AWS Glue supports 2 types of jobs
  * Spark/Pyspark
  * Python shell scripts
- Make sure to add a connection to REDSHIFT so that we can load the data to Redshift
  * Provide connection name in the Add connection in AWS Glue console
  * Select connection type as Amazon Redshift
  * Select cluster, database name, username and password and at last click finish
  * After that select the IAM, we need to create new IAM policy and select AWSGlueServiceRole, AmazonRedshiftFullAccess and AWSGlueConsoleFullAccess as role under AWS Service as Glue
- We're going to use Python shell jobs and the library which we're going to use is PyGreSQL

## AWS Datapipeline
- AWS Data pipeline is a service that helps you reliably process and move data between different AWS compute and storage services. It is a managed service that runs on EC2 instances under the hood
- We will create two data pipelines for dumping data from RDS MYSQL to S3. We'll import data from all the table and each of them will have different folder
  * Historical data folder under particular table, select run on pipeline activation(runs only one time)
  * Current data folder under particular table for incremental load

## AWS Datapipeline - Setup first hourly jobs for incremental data loads
- Schedule the job on HOURLY basis i.e. it will run after every one hour
- We're dumping data from the transactional system of last 3 months
- We will do this for all the query which we create after applying business logic and load it into S3 bucket with different folders and subfolders
  
  S3BucketName -> Folder(table_name) -> Current and Historical folder
   
## AWS Glue - Python shell job for incremental data load into redshift
- Python shell script, used boto3 and DB libraries, used Secret Manager for calling it for credentials instead of hard coding username and password
- Firstly we'll copy the current data from S3 to staging area(mysql_dwh_staging.orders) in Redshift, then I will delete all data the from the redshift final area(mysql_dwh.orders) where order id of the staging area and final area mathces so that we don't have to worry about the duplicacy and all. 
Once it's done, we'll load all the data into final area(mysql_dwh.orders) from the staging area (mysql_dwh_staging.orders). At the end, I will truncate the staging area table.

S3 to staging area -> Delete records from final area by matching -> Load data from Staging to final area -> Truncate the staging area datawarehouse

- Once it's done go to AWS Glue and select jobs and add a job (paste the incremental load python shell script while configuring)

## AWS Lambda Function to trigger our Glue job
- Above, we were manually running the incremental job in AWS Glue which is not feasible, so I created a lamda function(trigger_orders_hourly.py) which will trigger Glue job as soon as data loads into the S3 bucket from the AWS Data pipeline which runs on hourly basis.
- Used boto 3 and used 'Glue' as a cliet and provided job name(which we create in Glue). Also created new IAM role and attached policy - AWSLambdaExecute and AWSGlueConsoleFullAccess
- 






## Imp techniques used for loading data from RDS to DW
- We have CUSTOMER and ORDER table and they have one to one relationship(customer_id and order_id), so we'll join these tables and and store them in one table in Data warehouse. This will help in avoding joins in the data warehouse which will reduce the load on CPU, we can also accomodate two table from our transactional DB into one table in DW (This is making the use of columnar structure of Redshift)
- In ORDER and PRODUCT table we have one to one relationship(product_id and product_id), so we'll join them and store in one table in Datawarehouse. We'll use one script for storing historical data and one for current data of last 3 months - In DW under one table we have information of product & order table
 * Note -It is always good approach to dump tables which are more frequently changed and less freqently changed separately in Data warehouse.
- 


## Questions
- Why not stored historical data in Redhshift directly? There are many issue while debugging if we stored directly to RS, that's why using S3
- Why not done incremental copy of RDS MYSQL TO S3 in historical data dump? Because of the sync issue












