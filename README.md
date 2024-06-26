# AWS Glue streaming application to process Amazon MSK data using AWS Glue Schema Registry

## About Setup
In this solution, we will walk you through the steps to create and control the schema in Glue schema registry, integrate it with Java based application for data ingestion into MSK cluster, create a Glue streaming job for data extraction and processing from MSK cluster using schema registry, store processed data to Amazon s3 and query it using Amazon Athena 
The cloud infrastructure comprises of Amazon MSK cluster with two broker node across two subnet(availability zones), AWS Secret manager for Amazon MSK SASL/SCRAM based authentication, AWS Glue schema registry for hosting the schema, AWS Glue database and table of corresponding schema, AWS Glue Streaming job for processing the data on MSK cluster topics, and Amazon EC2 instance to build and run producer application to publish the data on MSK topics. The resources listed in the prerequisite section need to be available before proceeding with this solution.

CloudFormation template 'vpc-subnet-and-mskclient.template' will create a VPC with two private and two public subnets, SecurityGroups, Interface and VPC endpoints, S3 bucket for Glue job, IAM role and EC2 instance to run producer application. The purpose of public subnet is to create EC2 instance which can be reachable from the local machine to build and run producer Application. Another CloudFormation template `amazon-msk-and-glue.template` will setup the Amazon MSK cluster, AWS Glue Schema Registry, Glue Database, Table, Streaming Job and Crawler.

## Architecture Diagram
![Amazon MSK Processing the AWS Glue](/Images/Archtype.png) 
## Project Structure 
```
    .
    ├── amazon_msk_producer                   # Producer application sample code.
    |   ├── src
    |   ├── target    
    |   └── pom.xml
    |
    ├── glue_job_script                       # AWS Glue straming job script 
    ├── Images                                # Architecture image
    ├── amazon-msk-and-glue.template          # Cloudformation Template for creation of resources such as VPC, Subnet, S3 bucket, Secret and EC2 instance 
    ├── vpc-subnet-and-mskclient.template     # Cloudformation Template for creation of Amazon MSK cluster and Glue resources.
    ├── LICENSE
    └── README.md
```

## Prerequisites
- Ensure that:
    - You have an AWS Account. If you haven't created yet, [follow the instruction](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
    - You have an IAM User or IAM Role with necessary permission to create the resource using CloudFormation template
    - Create and download a valid key into the local machine to SSH into EC2 Instance. For instructions, see [Create a key pair using Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair). 

## Getting started
### Launch vpc_subnet_and_mskclient Stack. 
1. Clone the repo
  ```bash
  git clone https://github.com/aws-samples/aws-glue-msk-with-schema-registry.git
  cd aws-glue-msk-with-schema-registry
  ```
2. Launch the cloudformation template `vpc-subnet-and-mskclient.template`. Pass values for parameters. 
3. Once stack creation is completed, get the bucket name from stack *output* section, and copy the Glue Script to S3 bucket  
  ```bash
  aws s3 cp glue_job_script/mskprocessing.py s3://<bucket-name>/
  ``` 
### Launch Amazon MSK and Glue Stack. 
1. Launch the cloudformation template `amazon-msk-and-glue.template` and pass values from the *output* of stack `vpc-subnet-and-mskclient` This will take around 15-20 minute to complete 

### Produce data to Amazon MSK and consume it from Glue Straming job. 
1. Get the public dns name for EC2 instance from  `output` section of stack `vpc-subnet-and-mskclient` created in step 2, and [SSH into your instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

2. Clone the repository, build the producer application and run it as below.

```bash
  cd amazon_msk_producer
  mvn clean package
  BROKERS={OUTPUT_VAL_OF_MSKBootstrapServers – Ref. Step 6 of the blog post}
  REGISTRY_NAME={VAL_OF_GlueSchemaRegistryName - Ref. Step 6 of the blog post}
  SCHEMA_NAME={OUTPUT_VAL_OF_SchemaName - Ref. Step 6 of the blog post}
  TOPIC_NAME="test"
  SECRET_ARN={OUTPUT_VAL_OF_SecretArn – Ref. Step 3 of the blog post}
  java -jar target/amazon_msk_producer-1.0-SNAPSHOT-jar-with-dependencies.jar -brokers $BROKERS -secretArn $SECRET_ARN -region us-east-1 -registryName $REGISTRY_NAME -schema $SCHEMA_NAME -topic $TOPIC_NAME -numRecords 10
```

**Argument Supported for Producer Application.**

  | Parameter | Description | Default Value |
  | :----------: | :-------------: | :---------------: |
  | `region` | Region name where the Schema registry exist | `""` |
  | `brokers` | Amazon MSK Brokers | `""` |
  | `registryName` | Amazon Glue Schema Registry Name | `test-schema-registry` |
  | `schema` | Schema that is register with Schema registry and will be used in Producer application  | `test_payload_schema` |
  | `secretArn` | SecretManager Arn for autheticating with Amazon MSK using SASL/SCRAM  | `""` |
  | `topic` | Topic Name for publishing the data into Amazon MSK | `test` |
  | `numRecords` | Number of records you want to publish into Amazon MSK Topic | `10` |

1. Once message get published into MSK topic. Get the Glue Straming job name, Crawler name, output S3 bucket from stack `amazon-msk-and-glue` *output*.
2. Start the job to process the data published into MSK topic
3. Check the processed data into S3 bucket under `<bucket-name>/output` prefix, and run the crawler.
4. Run the Athena query against the table created by Crawler.
  | region | Region name where the Schema registry exist | "" |
  | brokers | Amazon MSK Brokers | "" |
  | registryName | Amazon Glue Schema Registry Name | test-schema-registry |
  | schema | Schema that is register with Schema registry and will be used in Producer application  | test_payload_schema |
  | secretArn | SecretManager Arn for autheticating with Amazon MSK using SASL/SCRAM  | "" |
  | topic | Topic Name for publishing the data into Amazon MSK | test |
  | numRecords | Number of records you want to publish into Amazon MSK Topic | 10 |

3. Once message get published into MSK topic. Get the Glue Straming job name, Crawler name, output S3 bucket from stack `amazon-msk-and-glue` *output*.
4. Start the job to process the data published into MSK topic
 
5. Check the processed data into S3 bucket under `<bucket-name>/output` prefix, and run the crawler.
6. Run the Athena query against the table created by Crawler.

# Security
See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.
# License
This library is licensed under the MIT-0 License. See the LICENSE file.

-----------------------------
aws s3 cp glue_job_script/mskprocessing.py s3://aws-glue-gluemsk-XXXXXX-us-east-1/
#Get bootstrp server name
aws kafka get-bootstrap-brokers --cluster-arn arn:aws:kafka:us-east-1:XXXXXXXXXX:cluster/amazon-msk-and-glue/aafe8b07-8a06-4f78-a190-cdf82dcf5ebf-14

#Java commad:
java -jar target/amazon_msk_producer-1.0-SNAPSHOT-jar-with-dependencies.jar -brokers  v-2.xxxx.yyyy.zz.kafka.us-east-1.amazonaws.com:9096   -secretArn AmazonMSK_vpc-subnet-and-mskclient	 -region us-east-1 -registryName test-schema-registry -schema test_payload_schema -topic test -numRecords 10



