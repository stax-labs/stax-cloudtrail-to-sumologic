# Stax Cloudtrail Logs to SumoLogic

- [Why?](#why)
- [Caveats](#caveats)
- [Deployment](#deployment)
  - [Configure Collector](#configure-collector)
  - [Deploy SumoLogic AWS Resources](#deploy-sumologic-aws-resources)
    - [Deploy Role](#deploy-role)
    - [Subscribe SNS to Collector](#subscribe-sns-to-collector)
      - [Console](#console)
    - [CLI](#cli)
  - [Addendum - Field Extraction](#addendum---field-extraction)

Deploys a simple SumoLogic (SL) forwarder and subscribes it to the Stax created cloudtrail SNS topic as documented on the Stax [Consuming AWS Service Logs](https://www.stax.io/help/kb/consuming-aws-service-logs/) documentation.

## Why?

Stax implements the [Organization Cloudtrail](https://docs.aws.amazon.com/organizations/latest/userguide/services-that-can-integrate-cloudtrail.html) on behalf of customers, with the log files for the trail ending up in the [Stax Logging Account](https://support.stax.io/hc/en-us/articles/4577859491471-Consume-AWS-Service-Logs-in-the-Logging-Account)

You could also follow the SumoLogic provided guide to setting up Cloudtrail when using Control Tower, however we found that this still requires manual tweaking (for example the SumoLogic created role doesn't allow access to KMS) and it leaves unused resources behind (SumoLogic creates it's own SNS topic which conflicts with teh Stax provided topic, there can be only one)

## Caveats

- This documentation is based on SumoLogic guidance for Control Tower setup, which also implements Organization Cloudtrail, it was correct at the time of writing, but changes to SumoLogic may render this solution in-operable. Please raise an issue on this repo if you run into problems and we'll do our best to assist.
- You will almost certainly incur costs in both SumoLogic and AWS when implementing this.
- This documentation is for the hosted collector only.

## Deployment

- AWS deployment steps assume you have a valid [AWS CLI](https://support.stax.io/hc/en-us/articles/4445073110159-Log-in-to-AWS-Accounts-Managed-by-Stax#link-4) session in your Stax Logging Account.
- SumoLogic collector configuration assumes you have Administrative access to SumoLogic

### Configure Collector

These instructions are a summary of the steps provided in the [AWS CloudTrail Source](https://help.sumologic.com/03Send-Data/Sources/02Sources-for-Hosted-Collectors/Amazon-Web-Services/AWS-CloudTrail-Source) guide that SumoLogic provides, if the steps below don't match what you see in the console, we strongly suggest reading the source document (and please create an Issue letting us know).

Additionally the role we've provided is functionally the same as the role that SumoLogic will generate for you in the collect setup, except that it will also provide access to the KMS key required to read from the bucket.

1. In Sumo Logic select Manage Data > Collection > Collection.
2. Select the hosted Collector for which you want to add the Source, and click Add Source (see [AWS Cloudtrail Source](https://help.sumologic.com/03Send-Data/Sources/02Sources-for-Hosted-Collectors/Amazon-Web-Services/AWS-CloudTrail-Source) for instructions on creating one).
3. Click AWS CloudTrail
4. Set the General Parameters:
   1. S3 Region: `Other`
   2. Bucket Name: this will be in the format `stax-cloudtrail-<org-uuid>` you can read more about it in our [docs](https://support.stax.io/hc/en-us/articles/4577859491471-Consume-AWS-Service-Logs-in-the-Logging-Account)
   3. Path Expression: `AWSLogs/*/CloudTrail/*`
   4. Source Category: `aws/observability/cloudtrail/logs`
   5. Fields: `account` `logging`
5. In the AWS Access section choose "Role-based access` and take a note of the "Account ID" and "External ID"
   1. **At this point we need to deploy the [role](#deploy-role)**
   2. Once the role is deployed, take the output ARN and provide it in the "Role ARN" field.
6. In the Log File discovery section click "Create URL"
   1. Select a 5 minute interval
   2. **At this point you need to deploy the [SNS Subscription](#subscribe-sns-to-collector)**
7. Remaining options are the defaults
8. When selecting save you might be told that the page is out of date and you need to refresh, after refreshing the source should still have been created.

### Deploy SumoLogic AWS Resources

The following steps explain what AWS resources you'll need to deploy.

#### Deploy Role

SumoLogic requires a role that the collector can assume which allows access to read files from the Cloudtrail S3 bucket, and access to the KMS Key used to encrypt data in that bucket, we have provided a template under the cloudformation folder in this repo, you can upload this through the console and set the Account ID and External ID based on the values generated during the [Configure Collector](#configure-collector) step or using the CLI:

You will need to get the ARN of the KMS Key that Stax uses to secure your Cloudtrail data, you can do this in the KMS console, by selecting the KMS Key with the alias "cloudtrail" or using the cli

```bash
aws kms list-aliases --query "Aliases[?AliasName=='alias/cloudtrail']"
```

Once you have this you can run the deployment:


```bash
aws cloudformation deploy \
    --template-file file://cloudformation/sumologic_role.yaml \
    --stack-name sumologic-stax-cloudtrail-role \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides \
        SumoLogicAccountID=<Account ID generated during collector setup> \
        SumoLogicExternalID=<External ID generated during collector setup> \
        StaxCloudtrailBucket=<The Stax Cloudtrail bucket, the same value provided to Sumo during collector setup> \
        StaxCloudtrailKMSKey=<The full ARN of the cloudtrail KMS key>
```

The output from this cloudformation deployment will be the ARN of the created role, you'll need this value to complete the collector setup.

#### Subscribe SNS to Collector

AWS only allows a single SNS notification on each event type on a bucket, since Stax has already generated this topic for you, there's no need to create your own, you just need to subscribe SumoLogic to the topic, you can configure this in the console or the AWS CLI.

The Stax created SNS topic will have a name that takes the format `cloudtrail-<stax-org-id>` you can find it in the SNS console in your logging account.

##### Console

Go to Services > Simple Notification Service and select the cloudtrail topic created by Stax.

- Click Create Subscription.
- Select HTTPS as the protocol
- Enter the Endpoint URL provided while [configuring the collector](#configure-collector) in Sumo Logic.
- Click Create subscription and a confirmation request will be sent to Sumo Logic. The request will be automatically confirmed by Sumo Logic.

Source: <https://help.sumologic.com/03Send-Data/Sources/02Sources-for-Hosted-Collectors/Amazon-Web-Services/AWS_Sources#set-up-sns-in-aws-highly-recommended>

#### CLI

```bash
aws sns subscribe --protocol https \
    --topic-arn <arn:aws:sns:<REGION>:<ACCOUNT_ID>:cloudtrail-<STAX-ORG-ID>
    --notification-endpoint <ENDPOINT URL>
```

### Addendum - Field Extraction

The SumoLogic AWS Observability for AWS Control Tower [guide](https://d1.awsstatic.com/Marketplace/solutions-center/downloads/SumoLogic-AWS-ControlTower-Implementation%20Guide-v2.0.pdf) also recommends setting up field extractions for your account IDs, see page 12 of the linked document for detailed instructions.
