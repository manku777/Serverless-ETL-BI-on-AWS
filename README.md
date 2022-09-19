# Serverless-ETL-BI-on-AWS

## PROBLEM STATEMENT

![image](https://user-images.githubusercontent.com/87073324/190860423-1bc878d8-9ced-4290-8aac-bf50437d85a3.png)



1. END POINTS are used to connect to RDS instance remotely, so make a note of it. So go to my SQL workbench or anything and click in "New Database Connection". End point shoudl be the host name while connecting, port will be 3306, also provide username and password. 
2. AWS Crawlers are used to connect to our datasources which can be csv, parquet etc and extracts the schema from it and populates the data catalog with the metadata from this file. In other term, it scan the sources, creates the table in DATA CATALOG which can be queried from Athena  


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
- Used boto 3 and used 'Glue' as a client and provided job name(which we create in AWS Glue). Also created new IAM role and attached policy - AWSLambdaExecute and AWSGlueConsoleFullAccess

## AWS Glue Crawler Setup
-  We're going to use this for 
 * Synced transactional data with Redshify
 * Data enrichment and data centralisation
- Create a S3 bucket(dwh_external_data) and create folder(user_behavior) in it and upload the 2016_funnel.csv
- Once we have file partitioned by 'year' & 'month' and uploaded in the S3 bucket in parquet format, we will again create the crawler and extract the parquet file and will compare with the csv created earlier in Athena   
 
## Pyspark Development Local
- Here we create a Pspark script which will extract the data from the S3(funnel.csv), apply transformation to the funnel data and will save them as parquet file format. Will also apply compression and partitioning to the data and store in Local. After that we will be uploading it in the cloud and create Glue job
- Extract the 'year' and 'month' from the eventtimestamp column and created two new columns i.e. year and month respectively and done partition on it

## AWS Lambda to Trigger Glue jobs
- Create AWS Lambda function which will take the name of the file(2016_funnel) and will do tranformation for that only. Provide the argument name in pyspark script as well(argument name in lamda function and pyspark job should be same). This will help in reducing the redundancy, for example applying transformation to same file again and again. 

## Redshift spectrum (Data centralization - Centralize data stored in Athen(funnel) into Redshift(transactional))
- For this we will create external schema in Redshift from the schema stored in data catalog (Athena). Database name in catalog newly created schema should be same.
- It will move the table from Athena to Redshift so that we can join table which is already in Redshfit with others and do transformation etc


## Visualization 
- Connect redshift cluster(order table under mysql_dwh) with the quicksight 
- Firstly create a schema in Redhshift and then do configuration in Quicksight like provide the schema name(quicksight analytics), view name(sales by category)  
  *  Visulaized the product category of each year and the number of time it got sold
  *  Joined the sales by category table with product category
  *  Funnel data has "event_type" like remove_from_cart, add_to_cart, view etc.. and also has column product_id, user_id etc... so I joined this table with customer table and visualize particular customer is removing products from the cart, viewing but not buying etc...

## Redshift optimization technique and tuning




## Imp techniques used for loading data from RDS to DW
- We have CUSTOMER and ORDER table and they have one to one relationship(customer_id and order_id), so we'll join these tables and and store them in one table in Data warehouse. This will help in avoding joins in the data warehouse which will reduce the load on CPU, we can also accomodate two table from our transactional DB into one table in DW (This is making the use of columnar structure of Redshift)
- In ORDER and PRODUCT table we have one to one relationship(product_id and product_id), so we'll join them and store in one table in Datawarehouse. We'll use one script for storing historical data and one for current data of last 3 months - In DW under one table we have information of product & order table
 * Note -It is always good approach to dump tables which are more frequently changed and less freqently changed separately in Data warehouse.
- 


## Questions/Challenges
- Why not stored historical data in Redhshift directly? There are many issue while debugging if we stored directly to RS, that's why using S3
- Why not done incremental copy of RDS MYSQL TO S3 in historical data dump? Because of the sync issue
- The funnel data file column 'timestamp' was in string format, so converted into timestamp in local(Jupyter). Extracted the 'year' and 'month from the eventtimestamp column and created two new columns i.e. year and month respectively and done partition on both columns
- Initally when I used crawler to load data from S3 bucket to data catalog in csv format and ran query using Athena, there was performance issue and was slow and same(same execution time) because csv is flat file. 
So to remove this performance issue and for partitioning, I created Glue Pyspark job which by:
 * Read the csv file(funnel.csv) from our local system and will apply basic transformation in it and will store as parquet in local. Once we're sure and fine, we will create a AWS Glue Pyspark job and attached a pyspark script in it which will load the data to S3. After this we will use crawler to extract the schema of the parquet file and will load it into data catalog which can be queried using Athena. There is major performance diffence b/w the parquet file and csv i.e. flat file uploaded in the data catalog
 * How lambda function knows whether the S3 bucket has been updated? We add a trigger in lamda function while configuring. As soon as there is an event(All object create events), it will trigger lambda function, which in turn will trigger AWS Glue Python shell job.

# Notes
 * Redshift spectrum: Used for centralizing the data(we have user behavioral data which is in Athena and transactional on Redshift)


# Summary

- I have schema and tables with inserted value in my local DBeaver. My first task was to move all the transactional data to RDS MYSQL. I used the below command for it and dumped into RDS MYSQL
  LOAD DATA LOCAL INFILE '/your_local_system_path/table_name.csv' 
  INTO TABLE table_name FIELDS TERMINATED BY ','
  ENCLOSED BY '"' IGNORE 1 LINES;
  
- Now I created two data pipeline to import the historical(one time) and last three months of data to S3 bucket in csv format. I create two folders i.e. historical and current in the S3 bucket which I will import it to Redshift(mysql_dwh.orders and mysql_historical_dwh). I joined customer and orders table on basis of common customerid.
- For loading the data from S3 bucket to Redshift make sure you have create the table in Redshift with exact columns in sequence order as we have in our csv file resding under S3 bucket
    copy schema.tablename from 's3://mysql-dwh/orders/historical/orders.csv'                  
   iam_role: 'arn:aws:iam::432432432:role/sdfsdf'                           // can get from the iam role which we created for Redshift
   CSV QUOTE '\"' DELIMITER ','
   acceptinvchars;                                                         // if any special chars, convert it into character  
- Now my task was to create a pipeline which will run on hourly basis and load the last 3 months of data into S3 bucket under current folder like we did above for historical data. SQL query which we'll mention while creating the data pipeline will be same except WHERE clause part.

- Once the historical and current data(hourly pipeline) has been created. I created the AWS Glue Python shell job(where I added python shell script while configuring) for incremental data load into redshift.
  * Python shell script, used boto3 and DB libraries, used Secret Manager for calling it for credentials instead of hard coding username and password
  * Firstly we'll copy the current data from S3 to staging area(mysql_dwh_staging.orders) in Redshift, then I will delete all data the from the redshift final area(mysql_dwh.orders) where ORDER ID of the staging area and ORDER ID of final area mathces so that we don't have to worry about the duplicacy and all. 
Once it's done, we'll load all the data into final area(mysql_dwh.orders) from the staging area (mysql_dwh_staging.orders). At the end, I will truncate the staging area table.

S3 to staging area -> Delete records from final area by matching -> Load data from Staging to final area -> Truncate the staging area datawarehouse

- Now we have created a incremental job but we can't run this manually every hour, so I created a lambda function which will trigger the incremental load job whenever there is an update in the current.csv file residing in S3. We need not to automate AWS datapipeline job because they run automatically after every hour.
 * Used boto 3 and 'Glue' as a client and provided job name(which we create in AWS Glue). Also created new IAM role and attached policy - AWSLambdaExecute and AWSGlueConsoleFullAccess
 * How lambda function knows whether the S3 bucket has been updated? We add a trigger in lamda function while configuring. As soon as there is an event(All object create events), it will trigger lambda function, which in turn will trigger AWS Glue Python shell job.

- Now we are dealing with funnel data. It is a type of data which is sent by third party like Amplitude where they analyze the user behaviour like adding to cart, buying, how frequently user is buying etc so that company can improve their service.
So this funnel data is is stored in our S3 bucket in csv format

- Now we will create one folder(user_behaviour) in the S3 bucket and will store the funnel data csv file and will Crawler which will extract the schema and load it exactly in Data Catalog which can be queried using Athena. While quering the dumped csv file in data catalog and quering from Athena the execution time was long and same even if the query is short. So to remove this, I used Pyspark(I did everything in loca jupyter notebook and upload parquet file in AWS).
 * Note - We will be first do in local and then mimic the same in AWS Glue PySpark
 * Here we create a Pspark script which will extract the data from the S3(funnel.csv), apply transformation to the funnel data and will save them as parquet file format. Will also apply compression and partitioning to the data and store in Local. After that we will be mimicing the local script procedure and perform the same in AWS Glue PySpark which will read the data from the S3(2016_funnel.csv)uploading it in the S3 after transformation and created a crawler which will read the schema, in our case parquet file with partition and create the same in data catalog
 * Extract the 'year' and 'month' from the eventtimestamp column and created two new columns i.e. year and month respectively and done partition on it

 * WE HAVE MADE THE SCRIPT DYNAMIC(by applying logic in filename) WHICH MEANS IT WILL PERFORM ETL ON ALL THE FILE WHICH ARE PRESENT IN S3 INSTEAD OF HARDCODING IT IN THE SCRIPT AND AFTER ETL IT WILL STORE IT IN S3 BUCKET. Now we will create the crawler as we did earlier and will load the parition data into Data Catalog

- Now we have USER BEHAVIOUR(FUNNEL DATA) in our ATHENA and transactional data in the REDSHIFT and we will be centralizing both of them with REDSHFIT SPECTRUM 








