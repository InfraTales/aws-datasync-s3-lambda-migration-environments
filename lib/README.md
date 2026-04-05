# Migration Pipeline Infrastructure

This project deploys a complete end-to-end migration pipeline on AWS using CDK (Cloud Development Kit). The infrastructure handles database migration, file synchronization, data validation, and orchestration of the entire migration process.

## What This Pipeline Does

The migration pipeline is designed to move data from an on-premises environment to AWS. It handles three main types of data:

1. Database data - migrates relational database content from on-premises MySQL to Aurora MySQL
2. File data - synchronizes files from NFS storage to S3
3. Data processing - validates and transforms data using AWS Glue ETL jobs

The pipeline includes monitoring, logging, and automated orchestration so the migration can run with minimal manual intervention.

## Architecture Overview

The infrastructure is organized into 11 nested stacks that work together:

**Network Layer** - Creates the VPC with public, private, and isolated subnets across two availability zones. Includes security groups for each service to control traffic flow.

**Storage Layer** - Sets up two S3 buckets: one for incoming data files and another for Glue scripts. Both buckets use encryption and versioning.

**Database Layer** - Deploys an Aurora MySQL cluster with one writer and one reader instance. The database is placed in isolated subnets and uses encryption at rest.

**Glue ETL Layer** - Creates Glue jobs for data validation and a Glue Data Catalog database to store metadata about the migrated data.

**Messaging Layer** - Sets up SNS topics for validation results and notifications about the migration status.

**DMS Layer** - Configures AWS Database Migration Service to handle the actual database replication from the source MySQL to Aurora. Includes endpoints for both source and target databases.

**Lambda Layer** - Deploys three Lambda functions:
- One triggers Glue jobs when files arrive in S3
- One starts the Step Functions workflow after validation
- One handles remediation when issues are detected

**DataSync Layer** - Launches an EC2 instance running the DataSync agent and creates a DataSync task to sync files from NFS to S3. The agent activation happens automatically through a custom resource.

**Orchestration Layer** - Uses AWS Step Functions to coordinate the migration workflow. The state machine starts DMS replication, waits for completion, checks status, and sends notifications.

**Monitoring Layer** - Sets up CloudWatch alarms and metrics to track the health of the migration pipeline.

**Logging Layer** - Deploys an OpenSearch domain for centralized log aggregation and analysis.

## How the Components Work Together

When you upload a file to the on-premises NFS share, DataSync detects it and syncs it to S3. The S3 upload triggers an EventBridge rule that invokes the Glue trigger Lambda. This Lambda starts a Glue job to validate the data quality.

Meanwhile, DMS is continuously replicating database changes from the source MySQL to Aurora. When the Glue validation completes, it publishes results to an SNS topic. This triggers another Lambda that starts the Step Functions state machine.

The state machine orchestrates the final steps of the migration process, checking that everything completed successfully and sending notifications. All logs from every service flow into the OpenSearch domain for centralized monitoring.

## Prerequisites

Before deploying this infrastructure, you need a few things set up:

**AWS Account** - You need an AWS account with sufficient permissions to create all the resources in this stack.

**AWS CLI Installed** - Download and install the AWS CLI from the official AWS website.

**Node.js and npm** - The CDK requires Node.js version 14 or later. Install it from nodejs.org.

**AWS CDK Installed** - Install the CDK toolkit globally:
```
npm install -g aws-cdk
```

**DMS IAM Role** - AWS DMS requires a special IAM role that must exist before you can create DMS resources. Create it once per account:

```
aws iam create-role \
  --role-name dms-vpc-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "dms.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
```

## Setting Up AWS Credentials

The CDK needs access to your AWS account to deploy resources. There are several ways to configure credentials:

**Method 1: Environment Variables**

Set these environment variables in your terminal:
```
export AWS_ACCESS_KEY_ID=your-access-key-id
export AWS_SECRET_ACCESS_KEY=your-secret-access-key
export AWS_DEFAULT_REGION=us-west-2
```

**Method 2: AWS CLI Configuration**

Run the AWS configure command and enter your credentials when prompted:
```
aws configure
```

This stores your credentials in ~/.aws/credentials and your default region in ~/.aws/config.

**Method 3: IAM Role (for EC2 or other AWS services)**

If you're running the deployment from an EC2 instance, you can attach an IAM role with the necessary permissions instead of using access keys.

Make sure your credentials have administrator access or at least permissions to create VPCs, EC2 instances, RDS clusters, Lambda functions, S3 buckets, IAM roles, and all the other services in this stack.

## Configuration

Before deploying, you may want to adjust some settings in the code:

**Environment Suffix** - The stack uses an environment suffix to make resource names unique. You can set it when deploying:
```
export ENVIRONMENT_SUFFIX=prod
```
Or modify the default in the code (currently defaults to 'dev').

**NFS Server IP** - In the DataSync stack, update the NFS server hostname from the placeholder IP address to your actual NFS server:
```typescript
serverHostname: '10.0.0.100'  // Change this to your real NFS server IP
```

**Database Credentials** - The Aurora cluster generates a random password stored in AWS Secrets Manager. If you want to use specific credentials, you'll need to modify the database stack.

**DMS Source Endpoint** - Update the source endpoint configuration with your actual on-premises database details:
```typescript
serverName: 'source.example.com'  // Your source database hostname
port: 3306
username: 'admin'
password: 'placeholder'  // Use Secrets Manager in production
databaseName: 'sourcedb'
```

## Deployment Steps

Once you have everything configured, follow these steps to deploy:

1. Clone the repository and navigate to the project directory

2. Install dependencies:
```
npm install
```

3. Bootstrap CDK in your AWS account (only needed once per account/region):
```
cdk bootstrap aws://ACCOUNT-ID/us-west-2
```

4. Review what will be deployed:
```
cdk synth
```

5. Deploy the stack:
```
cdk deploy --all --require-approval never
```

The deployment takes about 15-20 minutes. You'll see progress as each nested stack is created. The DataSync stack takes the longest because it needs to launch an EC2 instance and wait for the agent to start.

If deployment fails, check the error message in the output. Common issues include:
- Missing DMS IAM role (see Prerequisites)
- Insufficient IAM permissions
- Service quota limits reached
- Network connectivity issues


## After Deployment

Once the stack deploys successfully, you'll see outputs with important information:

- VPC ID
- S3 bucket names
- Aurora cluster endpoint
- OpenSearch domain endpoint
- Step Functions state machine ARN
- Lambda function ARNs

You can find these outputs in the CloudFormation console or by running:
```
aws cloudformation describe-stacks --stack-name TapStackpr5067
```

The DataSync task is scheduled to run daily at 2 AM UTC. You can also trigger it manually from the DataSync console.

The DMS replication runs continuously in CDC (change data capture) mode after the initial full load completes.

## Monitoring the Pipeline

All services send logs to CloudWatch Logs. The OpenSearch domain provides a centralized place to search and analyze logs.

CloudWatch alarms monitor:
- DataSync agent status
- DMS replication task health
- Lambda function errors
- Step Functions execution failures

SNS topics send notifications when validation completes or issues are detected.

## Cleaning Up

To avoid ongoing charges, delete the stack when you're done:

```
cdk destroy --all
```

This removes all resources except those with a RETAIN removal policy (like production databases and log groups). You may need to manually delete S3 buckets if they contain objects.

## Cost Considerations

The main costs come from:
- EC2 instance for DataSync agent (m5.large running 24/7)
- Aurora cluster (2 r6g.large instances)
- DMS replication instance (t3.medium)
- OpenSearch domain (t3.small.search instance)
- Data transfer between services
- S3 storage and requests

For development environments, consider using smaller instance sizes and stopping resources when not in use. The code uses cost-optimized instance types where possible.

## Troubleshooting

**DataSync agent won't activate**
The Lambda function retries for up to 9 minutes. If it still fails, check that:
- The EC2 instance launched successfully
- Security groups allow HTTP traffic on port 80
- The agent service started (check EC2 console system logs)

**DMS replication fails**
Common causes:
- Source endpoint can't be reached from the VPC
- Incorrect database credentials
- Firewall blocking connection
- Binary logging not enabled on source MySQL

**Glue jobs fail**
Check CloudWatch Logs for the Glue job to see the specific error. Often related to:
- Invalid S3 paths
- Missing IAM permissions
- Python script errors

**Step Functions execution fails**
Look at the state machine execution history in the Step Functions console. Each step shows whether it succeeded or failed and why.


The infrastructure is defined in tap-stack.ts using nested stack classes. Each stack class handles one layer of the architecture and exposes properties needed by other stacks.

## Security Notes

This infrastructure follows AWS security best practices:
- Resources are placed in private subnets when possible
- Security groups restrict traffic to only what's needed
- Encryption is enabled for data at rest and in transit
- IAM roles follow least privilege principle
- VPC flow logs can be enabled for network monitoring

For production use, consider additional hardening:
- Enable MFA for AWS account access
- Use AWS Secrets Manager for all credentials
- Enable CloudTrail for audit logging
- Implement backup and disaster recovery procedures
- Use AWS Config to monitor compliance

## Future Enhancements

Possible improvements to this pipeline:
- Add data quality checks with AWS Glue DataBrew
- Implement automatic rollback on validation failures
- Use AWS EventBridge Scheduler for more flexible scheduling
- Add support for multiple source databases
- Implement blue/green deployment for database migrations
- Use AWS Backup for automated backups
- Add cost optimization with EC2 Spot instances where appropriate

## Support

For issues related to AWS services, consult the AWS documentation or AWS Support.

For issues with this CDK code, check the AWS CDK documentation and GitHub issues.

## License

This infrastructure code is provided as-is for migration purposes.