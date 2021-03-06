# Data Lake S3 triggered staging + data catalog engine

  ![image text](https://raw.githubusercontent.com/andresmaopal/data-lake-staging-engine/master/Resources/diagrama_arquitectura_small.png)

A packaged Data Lake solution to partition the data in a Raw bucket, copy the datasets in parquet on an Staging bucket, Sync the Glue Catalog Databases, Glue Crawlers and Data Stores building a highly functional Data Lake, with a complimentary metadata Data catalog queryable via Elasticsearch (optional)

As its designed for multi-country support, the expected naming convention for the data lake is:

`octank-dev-raw/landing/country_code/database_name/schema_name/table_name/` where `landing/` is the folder where the ingestion service (like AWS Database Migration Service) will be writing.


## License

This library is licensed under the Apache 2.0 License. 

## Staging engine DataLake installation general instructions

These are the steps required to provision the  Packaged Datalake Solution and watch the ingress of data.
* Provision the Data Lake Structure (5 minutes)
* Provision the Visualisation
    * Provision Elasticsearch (15 minutes)
    * Provision Visualisation Lambdas (5 minutes)
* Provision the Staging Engine and add a trigger (10 minutes)
* Configure a sample data source and add data (5 minutes)
    * Configure the sample data source
    * Ingress a sample file for the new data source
    * Ingress a sample file that has an incorrect schema
* Initialise and use Elasticsearch / Kibana (5 minutes)

Once these steps are complete, the provided sample data file (or any file matching the sample data source criteria) can be dropped in the DataLake's raw bucket and will be immediately staged and recorded in the data catalog.

If a file dropped into the raw bucket fails staging (for example it has an incorrect schema, or non-recognised file name structure), it will be moved to the failed bucket and recorded in the data catalog.

Whether staging is successful or not, the file's ingress details can be seen in the datalake's elasticsearch / kibana service.

NOTE: There are clearly improvements that can be made to this documentation and to the cloudformation templates. These are being actioned now. 

## 1. Provisioning the Data Lake Structure
This section creates the basic structure of the datalake, primarily the S3 buckets and the DynamoDB tables. 

Execution steps:
* Go to the CloudFormation section of the AWS Console.
* Think of an environment prefix for your datalake. This prefix will make your S3 buckets globally unique (so it must be lower case) and wil help identify your datalake components if multiple datalakes share an account (not recommended, the number of resources will lead to confusion and pottential security holes). Ideally the prefix should contain the datalake owner / service and the environment - a good example is: wildrydes-dev-
* Create a new stack using the template `/DataLakeStructure/dataLakeStructure.yaml` 
* Enter the stack name. For example: `octank-dev-datalake-structure`
* Enter the environment prefix, in this case: `octank-dev-`
* Add a KMS Key ARN if you want your S3 Buckets encrypted (recommended - also, there are further improvements with other encryption options imminent in this area)
* All other options are self explanatory, and the defaults are acceptable when testing the solution.

## 2. Provisioning the Visualisations (Optional)
This step is optional, but highly recommended. If the customer does not want elasticsearch, a temporary cluster will allow debugging while the datalake is set up, and will illustrate its value. 

If you want elasticsearch visualisation, both of the following steps are required (in order).

### 2.1 Provision Elasticsearch
This step creates the Elasticsearch cluster the datalake will use. For dev databases, a single T2 instance is acceptable. For production instances, standard elasticsearch best practise apply.

NOTE: Elasticsearch is a very easy service to over-provision. To assist in determining the correct resources, the cloudformation template includes a CloudWatch dashboard to display all relevant cluster metrics.

Execution steps:
* Go to the CloudFormation section of the AWS Console.
* Create a new stack using the template `/Visualisation/elasticsearch/elasticsearch.yaml`
* Enter the stack name. For example: `octank-dev-datalake-elasticsearch`
* Enter the environment prefix, in this case: `octank-dev-`
* Enter the your ip addresses (comma separated), in this case: `<your_ip/32>,<another_cidr>`
* Change the other parameters as per requirements / best practise. The default values will provision a single instance `t2.medium` node - this is adequate for low tps dev and testing.

### 2.2 Provision the Visualisation Lambdas
This step creates a lambda which is triggered by changes to the data catalog DynamoDB table. The lambda takes the changes and sends them to the Elasticsearch cluster created above. 

Execution steps:
* Create a data lake IAM user, with CLI access.
* Configure the AWS CLI with the user's access key and secret access key.
* Install AWS SAM.
* Open a terminal / command line and move to the Visualisation/lambdas/ folder
* Package and deploy the lambda functions. There are following two ways to deploy it:
	* Execute the ``./deploy.sh <environment_prefix>`` script OR
	* Execute the AWS SAM package and deploy commands detailed in: deploy.txt

For this example, the commands should be:
````
sam package --template-file ./lambdaDeploy.yaml --output-template-file lambdaDeployCFN.yaml --s3-bucket octank-dev-visualisationcodepackages

sam deploy --template-file lambdaDeployCFN.yaml --stack-name octank-dev-datalake-elasticsearch-lambdas --capabilities CAPABILITY_IAM --parameter-overrides EnvironmentPrefix=wildrydes-dev-
````

## 3. Provision the Staging and Data Catalog engine and add an S3 trigger
This is the workhorse of the Staging engine - it creates lambdas and a step function, that takes new files dropped into the raw bucket, verifies their source and schema, applies tags and metadata, then copies the file to the staging bucket.

On both success and failure, Staging engine updates the DataCatalog table in DynamoDB. All changes to this table are sent to elasticsearch, allowing users to see the full history of all imports and see what input files were used in each DataLake query.

Execution steps:
(ignore these steps if already done in the visualisation step)
* Create a data lake IAM user, with CLI access.
* Configure the AWS CLI with the user's access key and secret access key.
* Install AWS SAM.
(mandatory steps: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
* Open a terminal / command line and move to the StagingEngine/ folder
* Package and deploy the lambda functions. There are following two ways to deploy it:
        * Execute the `./deploy.sh <environment_prefix>` script OR
        * Execute the AWS SAM package and deploy commands detailed in: `deploy.txt`

For this example, the commands should be:
````
sam package --template-file ./stagingEngine.yaml --output-template-file stagingEngineDeploy.yaml --s3-bucket octank-dev-stagingenginecodepackages

sam deploy --template-file stagingEngineDeploy.yaml --stack-name octank-dev-datalake-staging-engine --capabilities CAPABILITY_IAM --parameter-overrides EnvironmentPrefix=octank-dev-
````

### 3.1 Add the Staging trigger
Cloudwatch cannot create and trigger lambdas from changes to existing S3 buckets - this forces the Staging Engine startFileProcessing lambda to have its S3 PUT trigger attached manually.

Execution steps:
* Go into the AWS Console, Lambda screen.
* Find the lambda named: `<ENVIRONMENT_PREFIX>-StartFileProcessing-<RANDOM CHARS ADDED BY SAM>`
* Manually add two S3 triggers: 1) on PUT events, 2) on MULTIPART UPLOAD events selecting the RAW bucket (in this example, this would be `octank-dev-raw`) with the prefix: `landing/` on both triggers, in order to restrict them to this folder.

**NOTE:** Do not use "Object Created (All)" as a trigger - Staging-Catalog-Engine copies new files when it adds their metadata, so a trigger on All will cause the staging process to begin again after the copy.

### 3.2 Add a Lambda Layer to a Lambda who requires it

As this solutions moves each triggered file to a partitioned folder (on Raw) and makes a parquet copy (on Staging). An external Python library is required, so for this solution we are going to use [AWS Data Wrangler](https://github.com/awslabs/aws-data-wrangler/ ) which is a data utility belt with several ETL functions.

In order to do that we need to create a [Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) to attach it into the CopyFileFromRawToStaging Lambda. 

Execution steps:

* Go into the AWS Console, Lambda screen and click on "Layers"
* Create a new Layer with the name `aws_data_wrangler_0_0_23` and select "Upload a file from Amazon S3"
* Click this [link](https://github.com/awslabs/aws-data-wrangler/releases/download/0.0.23/awswrangler-layer-0.0.23-py3.6.zip) and save the .zip file and upload it to a temporal S3 bucket in your account or just copy this public path: https://public-slides-bucket.s3.amazonaws.com/lib/awswrangler-layer-0.0.23-py3.6.zip
* Choose "Python 3.6" as runtime

* Find the Lambda named: `<ENVIRONMENT_PREFIX>-CopyFileFromRawToStaging-<RANDOM CHARS ADDED BY SAM>` and in the "Layers" box add the previoulsy created Lambda Layer and Save the modified Lambda function.



Congratulations! The Staging engine is now fully provisioned! Now let's configure a datasource and add some data.

## 4. Configure a sample data source and add data
### 4.1 Configure the sample data source
Execution steps:
* Open the file `DataSources/sampleData/ddbDataSourceConfig.json`
* Copy the file's contents to the clipboard.
* Go into the AWS Console, DynamoDB screen.
* Open the DataSource table, which for the environment prefix used in this demonstration will be: `octank-dev-dataSources`
* Go to the Items tab, click Create Item, and paste in the contents of the file `ddbDataSourceConfig.json` located in /DataSources/sampleData/. 
* Save the created item.

You now have a fully configured DataSource. The individual config attributes will be explained in the next version of this documentation.

### 4.2 Ingress a sample file for the new data source
Execution steps:
* Go into the AWS Console, S3 screen, open the raw bucket (`octank-dev-raw` in this example)
* Create a new folder "landing" to be the folder to write data.
* Within the folder `landing` create new subfolders until have this path: "us/database_name/schema_name/orders".  
* Using the Web console, upload the file `DataSources/sampleData/LOAD0000001.csv` into this folder.
* Confirm the file has appeared in the Raw (partitioned folder) and  Staging folder, with a path similar in RAW like: `octank-dev-raw/partitioned/us/database_name/schema_name/orders/dt=2018-10-26/LOAD0000001.csv` and in STAGING: `octank-dev-staging/co/database_name/schema_name/table_name/dt=2018-10-26/89585bcb3984476fad1d028ba524d9dd.parquet`

If the file is not in the staging folder, one of the earlier steps has been executed incorrectly.
If the file is not in the failed folder, one of the earlier steps has been executed incorrectly.

## 5. Initialise and use Elasticsearch / Kibana
The above steps will have resulted in the rydebookings files being entered into the DynamoDB datacatalog table (`wildrydes2-dev-dataCatalog`). The visualisation steps subscribed to these table's stream and all updates are now sent to elasticsearch. 

The data will already be in elasticsearch, we just need to create a suitable index.

Execution steps:
* Go to the kibana url (found in the AWS Console, under elasticsearch)
* You will see there is no data - this is because the index needs to be created (the data is present, so we will let kibana auto-create it)
* Click on the management tab, on the left.
* Click "Index Patterns"
* Paste in: `octank-dev-datacatalog` (so `<ENVIRONMENT_PREFIX>datacatalog`). You will see this name in the available index patterns at the base of the screen.
* Click "Next step"
* Select `@Timestamp` in the "Time Filter field name" field - this is very important, otherwise you will not get the excellent kibana timeline.
* Click "Create Index Pattern" and the index will be created. Click on the Discover tab to see your data catalog and details of your failed and successful ingress. 

-----

This project is forked and customized based on [AWS Accelerated Data Lake](https://github.com/aws-samples/accelerated-data-lake)
