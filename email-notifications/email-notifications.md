# Content Commons Email Notifications

## Schematic

[View a visual schematic of the email notifications pipeline](https://app.mural.co/t/digitallab2105/m/digitallab2105/1634212286953/aa805a17c60277630f720845b112cc54e493577a).

## Email pipeline

### 1. Uploading the email recipients

The bulk email pipeline is initiated when the Commons Server uploads a JSON file to the `gpalab-email-recipients` S3 bucket. An examples of the JSON object for new playbooks is included below. Note that the values for `templateName` and `configurationSetName` must match the template name and SES configuration set in AWS. The JSON object for updated playbooks is the same, except the `templateName` value would instead be `UpdatedProjectTemplate`.

```
// for new playbooks
{
  "config": {
    "configurationSetName": "gpalab-ses-errors",
    "templateName": "NewProjectTemplate",
    "templateData": {
      "link": "https://commons.america.gov/playbooks/<project-title>",
      "title": "<Project Title>",
      "projectId": "<user-notification-id>"
    }
  },
  "recipients": [
    {
      "firstName": "<UserFirstName>",
      "email": "username@america.gov",
      "unsubscribe": "https://commons.america.gov/user/notification/update?us=playbook_publish_create&token=<unique-token>"
    },
    ...
  ]
}

```

Once the JSON file has been uploaded to the `gpalab-email-recipients` bucket, the `gpalab-chunk-email` Lambda function creates chunked JSON files with up to 500 email recipients per file.

For instance, if a JSON file has 2,000 recipients, the Lambda will create four separate chunk files with 500 recipients each (2,000 recipients / 500 recipients per chunk = four chunk files) and will write each chunk file to the `gpalab-email-recipients-chunks` S3 bucket.

#### 1.1 S3 settings for `gpalab-email-recipients` and `gpalab-email-recipients-chunks`
* Bucket and objects not public
* Default encryption enabled
* Server-side encryption: Amazon S3 master-key (SSE-S3)
* Tags:

| Key         | Value                            |
|:------------|:---------------------------------|
| Environment | gpalab-prod                      |
| Name        | gpalab-email-recipients[-chunks] |

#### 1.2 Lambda settings for `gpalab-chunk-email`

* Timeout: 15 seconds
* Trigger: gpalab-email-recipients
* VPC
    * k8s-prod
    * Subnets
        * us-east-1a, k8s-vpc-Subnet01
        * us-east-1c, k8s-vpc-Subnet03
    * Security groups, Inbound/Outbound rules
        * gpalab-lambda-prod
* Tags:

| Key         | Value                   |
|:------------|:------------------------|
| Environment | gpalab-prod             |
| Name        | gpalab-email-recipients |

#### 1.3 Creating, updating, and deleting SES templates

AWS has good documentation on [managing email templates](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-personalized-email-manage-templates.html) that describes how to create, update, view, and delete templates.

The templates should have the following shape. Note the Mustache style syntax for template values, e.g., `{{playbookTitle}}`. These values are set in the JSON file that gets uploaded with the recipients to the`gpalab-email-recipients-chunks` S3 bucket described earlier.

```
// template for new playbooks
{
  "Template": {
    "TemplateName": "NewProjectTemplate",
    "SubjectPart": "A new Playbook titled “{{playbookTitle}}” is now available!",
    "TextPart": "...",
    "HtmlPart": "..."
  }
}
```

### 2. Building the SES email parameters

When a chunked JSON file with email recipients is created in the `gpalab-email-recipients-chunks` bucket, the `gpalab-build-email` Lambda reads the chunked JSON file, builds the SES parameters for each recipient in the file, and sends an SQS message with the SES parameters for each recipient to the `gpalab-email-queue.fifo`.

Asynchronous failures of the `gpalab-build-email` Lambda are captured by `gpalab-email-lambda-destination-queue`, and `gpalab-email-queue.fifo` failures are captured by its dead-letter queue, `gpalab-email-queue-dlq.fifo`.

#### 2.1 Lambda settings for `gpalab-build-email`
* Timeout: 1 minute, 30 seconds
* Trigger: gpalab-email-recipients-chunks
* Destination (on failure): gpalab-email-lambda-destination-queue
* VPC
    * k8s-prod
    * Subnets
        * us-east-1a, k8s-vpc-Subnet01
        * us-east-1c, k8s-vpc-Subnet03
    * Security groups, Inbound/Outbound rules
        * gpalab-lambda-prod
* Tags:

| Key         | Value              |
|:------------|:-------------------|
| Environment | gpalab-prod        |
| Name        | gpalab-build-email |

#### 2.2 SQS settings for `gpalab-email-queue.fifo`

* Default visibility timeout: 15 minutes
* Message retention period: 14 days
* Receive message wait time: 20 seconds
* Lambda trigger: gpalab-send-email
* Dead-letter queue (see settings below): gpalab-email-queue-dlq.fifo
* Tags:

| Key         | Value                   |
|:------------|:------------------------|
| Environment | gpalab-prod             |
| Name        | gpalab-email-queue-fifo |

##### 2.2.1 SQS settings for `gpalab-email-queue-dlq.fifo`
* Maximum receives: 1,000
* Message retention period: 14 days
* Tags:

| Key         | Value                       |
|:------------|:----------------------------|
| Environment | gpalab-prod                 |
| Name        | gpalab-email-queue-dlq-fifo |

##### 2.2.2 SQS settings for `gpalab-email-lambda-destination-queue`
* Message retention period: 14 days
* Tags:

| Key         | Value                                 |
|:------------|:--------------------------------------|
| Environment | gpalab-prod                           |
| Name        | gpalab-email-lambda-destination-queue |

### 3. Sending SES emails to recipients

The `gpalab-send-email` Lambda is invoked when SQS messages appear in `gpalab-email-queue.fifo`, and it uses the SES parameters in the message to send an email to the recipient.

Because AWS has a rate limit of 14 emails per second, the `gpalab-send-email` implements a variation of the exponential backoff algorithm described in the AWS blog post, ["How to handle a 'Throttling – Maximum sending rate exceeded' error"](https://aws.amazon.com/blogs/messaging-and-targeting/how-to-handle-a-throttling-maximum-sending-rate-exceeded-error/).

If an email can't be sent successfully after the 10 attempts set in the exponential backoff algorithm, `gpalab-send-email` sends the message to the `gpalab-send-email-dlq` queue where it is picked up by the `gpalab-resend-email` Lambda and added back to the `gpalab-email-queue.fifo` queue for resending.

Asynchronous failures of the `gpalab-send-email` Lambda are captured by the `gpalab-email-lambda-destination-queue`.

#### 3.1 Lambda settings for `gpalab-send-email`
* Timeout: 1 minute, 30 seconds
* Trigger: gpalab-email-queue.fifo
    * Batch size: 1
* Destination (on failure): gpalab-email-lambda-destination-queue
* VPC
    * k8s-prod
    * Subnets
        * us-east-1a, Lambda-private-subnet
        * us-east-1c, lambda-private-subnet-2
    * Security groups, Inbound/Outbound rules
        * gpalab-lambda-prod
* Tags:

| Key         | Value             |
|:------------|:------------------|
| Environment | gpalab-prod       |
| Name        | gpalab-send-email |

#### 3.2 Lambda settings for `gpalab-resend-email`
* Timeout: 3 seconds
* Trigger: gpalab-send-email-dlq
    * Batch size: 1
* Destination (on failure): gpalab-email-lambda-destination-queue
* VPC
    * k8s-prod
    * Subnets
        * us-east-1a, k8s-vpc-Subnet01
        * us-east-1c, k8s-vpc-Subnet03
    * Security groups, Inbound/Outbound rules
        * gpalab-lambda-prod
* Tags:

| Key         | Value               |
|:------------|:--------------------|
| Environment | gpalab-prod         |
| Name        | gpalab-resend-email |

#### 3.2 SQS settings for `gpalab-send-email-dlq`
* Message retention period: 14 days
* Lambda trigger: gpalab-resend-email
* Encryption: AWS Key Management Service key (SSE-KMS)
* Tags:

| Key         | Value                 |
|:------------|:----------------------|
| Environment | gpalab-prod           |
| Name        | gpalab-send-email-dlq |

## Email errors pipeline

### 1. Capturing SES error events

SES error events are captured by the `gpalab-ses-errors` configuration set, which listens for the following events:

* Hard bounces,
* Complaints,
* Delivery delays,
* Rejects, and
* Rendering failures.

The SNS destination for these events is the `gpalab-ses-errors-topic`, which has the `gpalab-filter-ses-errors` Lambda as its only subscription.

#### 1.1 SNS settings for `gpalab-ses-errors-topic`

* Display name: Prod SES error
* Subscriptions
    * Lambda: gpalab-filter-ses-errors

### 2. Filtering the SES error events

After an SES error event is received from `gpalab-ses-errors-topic`, the `gpalab-filter-ses-errors` Lambda sends the error event object to one of the following SQS queues, depending on the type of error event:

| Event type            | SQS queue                                   |
|:----------------------|:--------------------------------------------|
| Bounce (Permanent)    | gpalab-ses-addresses-to-remove-queue.fifo   |
| Bounce (Undetermined) | gpalab-ses-auto-response-bounces-queue.fifo |
| Bounce (Transient)    | gpalab-ses-soft-bounces-queue.fifo          |
| Complaint             | gpalab-ses-addresses-to-remove-queue.fifo   |
| DeliveryDelay         | gpalab-ses-delivery-delays-queue.fifo       |
| Reject                | gpalab-ses-rejections-queue.fifo            |
| Rendering Failure     | gpalab-ses-rendering-failures-queue.fifo    |


#### 2.1 Lambda settings for `gpalab-filter-ses-errors`
* Timeout: 3 seconds
* Trigger: gpalab-ses-errors-topic
* VPC
    * k8s-prod
    * Subnets
        * us-east-1a, k8s-vpc-Subnet01
        * us-east-1c, k8s-vpc-Subnet03
    * Security groups, Inbound/Outbound rules
        * gpalab-lambda-prod

#### 2.2 SQS settings for all SES event error queues
* Type: FIFO
* Message retention period: 14 days
* Encryption: AWS Key Management Service key (SSE-KMS)
* Tags:

| Key         | Value                                 |
|:------------|:--------------------------------------|
| Environment | gpalab-prod                           |
| Name        | name of the queue without .fifo       |

#### 2.3 AWS resources on SES event publishing
* [Contents of event data that Amazon SES publishes to Amazon SNS](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/event-publishing-retrieving-sns-contents.html)
* [Examples of event data that Amazon SES publishes to Amazon SNS](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/event-publishing-retrieving-sns-examples.html)

### 3. Handling SES error events

SES error events are handled according to the specific queue its been routed to by the `gpalab-filter-ses-errors` Lambda.

For errors in the `gpalab-ses-addresses-to-remove-queue.fifo` queue, an SQS listener on the Commons server sets the email address status as "INACTIVE" in the Commons database. This prevents subsequent email notifications from being sent to email addresses that are marked as spam or are hard bounces. After an email address is received by the SQS listener on Commons server, it's removed from the `gpalab-ses-addresses-to-remove-queue.fifo` queue.

Errors in the `gpalab-ses-auto-response-bounces-queue.fifo` queue are not handled since they typically include ISP related events, which are beyond our scope of control.

However, the following four queues have CloudWatch alarms that forward to the `gpalab-ses-exceptions-notifications` SNS topic, which, in turn, sends a notification email to the system administrator:

| SQS queue                                   | CloudWatch alarm                    |
|:--------------------------------------------|:------------------------------------|
| gpalab-ses-soft-bounces-queue.fifo          | gpalab-ses-soft-bounces alarm       |
| gpalab-ses-delivery-delays-queue.fifo       | gpalab-ses-delivery-delays alarm    |
| gpalab-ses-rejections-queue.fifo            | gpalab-ses-rejections-queue alarm   |
| gpalab-ses-rendering-failures-queue.fifo    | gpalab-ses-rendering-failures alarm |

#### 3.1 CloudWatch alarm settings
* Metric name: ApproximateNumberOfMessagesVisible
* Threshold type: Static
* Threshold: > 0 for 1 datapoints within 1 minute
* Statistic: Sum



