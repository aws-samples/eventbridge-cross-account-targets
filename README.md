# Introducing cross-account targets for Amazon EventBridge event delivery rules

This repository is to support the AWS Compute Blog - [Introducing cross-account targets for Amazon EventBridge event delivery rules](https://aws.amazon.com/blogs/compute/introducing-cross-account-targets-for-amazon-eventbridge-event-buses/)

> **Important:** \_this application uses various AWS services and there are costs associated with these services after the Free Tier usage - please see the [AWS Pricing page](https://aws.amazon.com/pricing/) for details. You are responsible for any AWS costs incurred. No warranty is implied in this example.

## Sample architecture overview

This sample application demonstrates cross-account targets for Amazon EventBridge:

![Architecture diagram](/assets/1-cross-account-targets-diagram.png)

Please read the blog post to get additional information about this solution.

## Pre-requisites:

- [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- Install or update your AWS Command Line Interface (AWS CLI) to the [latest version](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
- Install or update your AWS Serverless Application Model (AWS SAM) CLI to the [latest version](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
- [Configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) AWS CLI to get access to two of your AWS accounts. If you do not have AWS accounts, please follow [instructions](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html) and create them.

We are going to use **source_eb_profile** and **destination_eb_profile** profile names for this instruction. Please replace them with your AWS accounts profile names when you run commands.

## Destination Stack deployment instructions:

1. Clone/Download this GitHub repository
2. Open a command-line tool of your preference, and navigate into the folder where you cloned or extracted this repository
3. Navigate into **destination-account** folder
   ```
   cd destination-account
   ```
4. Test your AWS SAM installation by running AWS SAM [validate](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-validate.html) command:

   ```
   sam validate --lint
   ```

   You should see a message:

   ![AWS SAM validate result](/assets/2-1-destination-stack-validation.png)

   > **Important:** _If you see any errors, please double check that you have AWS SAM installed properly, and you copied the template without changing indentation._

5. Deploy the destination stack to your configured destination AWS account by running AWS SAM [deploy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-deploy.html) command. Please replace **DESTINATION_ACCOUNT_PROFILE** with destination account AWS profile from your credentials file, **SOURCE_ACCOUNT_ID** with AWS source account id you configured prior and **EMAIL** with your email address:

   ```
   sam deploy --stack-name destination-eb-stack --capabilities CAPABILITY_IAM --profile DESTINATION_ACCOUNT_PROFILE --parameter-overrides SourceAccountId=SOURCE_ACCOUNT_ID SnsSubscriptionEmail=EMAIL
   ```

   By running this command, AWS SAM creates a deployment Amazon Simple Storage Service (Amazon S3) bucket within your AWS account in your configured AWS region. After that, it will create Amazon SQS queue, Amazon SNS Topic and AWS Lambda function with permissions for our source account.

   You should see a message after AWS SAM finishes the deployment:

   ![Image](/assets/2-destination-stack-deployment.png)

   > **Important:** _If you see any errors, please check that you properly [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) credentials for your AWS account and that you have enough permissions to run the deployment._

6. Please note **TargetSqsQueueArn**, **TargetSnsTopicArn** and **TargetLambdaFunctionArn** output parameters. We will need them for source stack deployment.

7. Open email client for the email address that you specified as a parameter to the deploy command and confirm SNS Topic subscription:

   ![Image](/assets/7-sns-topic-subscription.png)

## Source Stack deployment instructions:

1. Open **samconfig.toml** file in **source-account** folder and replace SAM parameters with destination stack output values you noted in the previous step:

   ![Image](/assets/3-sam-config-update.png)

2. Navigate into **source-account** folder:
   ```
   cd ../source-account
   ```
3. Test your AWS SAM installation by running AWS SAM [validate](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-validate.html) command:

   ```
   sam validate --lint
   ```

   You should see a message:

   ![AWS SAM validate result](/assets/2-2-source-stack-validation.png)

   > **Important:** _If you see any errors, please double check that you have AWS SAM installed properly, and you copied the template without changing indentation._

4. Replace **SOURCE_ACCOUNT_PROFILE** with source account AWS profile from your credentials file. Deploy source stack to your source account:

   ```
   sam deploy --stack-name source-eb-stack --capabilities CAPABILITY_IAM --profile SOURCE_ACCOUNT_PROFILE
   ```

   You should see a message after AWS SAM finishes the deployment:

   ![Image](/assets/3-1-source-account-deployment.png)

   > **Important:** _If you see any errors, please check that you properly [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) credentials for your AWS account and that you have enough permissions to run the deployment._

## Publishing a test message

1. Log-in into source account and open [EventBridge console](https://console.aws.amazon.com/events/home) in your browser.
2. Click on **Event buses** link:

   ![Image](/assets/8-event-bridge-message-publishing.png)

3. Click on **SourceEventBus** name:

   ![Image](/assets/8-1-event-bridge-message-publishing-event-bus.png)

4. Click on **Send events** button:

   ![Image](/assets/8-2-event-bridge-message-publishing-event-bus.png)

5. Set **Event source** to:
   ```
   demo.sqs
   ```
6. Set **Detail type** to:
   ```
   TestEvent
   ```
7. Set **Event detail** to:
   ```
   { "message":"Hello from the source account!"}
   ```
8. Click on **Send** button to publish the test event:

   ![Image](/assets/8-3-event-bridge-message-publishing-send.png)

## Testing cross-account SNS delivery

1. Check your mailbox, you should get an email from **AWS Notifications** within a few minutes:

   ![Image](/assets/7-1-sns-topic-delivery.png)

   > **Important:** _If you did not get an email, please double check you confirmed SNS topic subscription_

## Testing cross-account SQS delivery

1. Log-in into destination account and open [SQS console](https://console.aws.amazon.com/sqs/v2/home) in your browser.
2. Click on **destination-eb-stack-TargetSqsQueue-\*** name:

   ![Image](/assets/4-sqs-queue-console.png)

3. Click on **Send and receive messages** button:

   ![Image](/assets/5-sqs-send-receive.png)

4. Click on **Poll for messages** button:

   ![Image](/assets/6-sqs-poll-for-messages.png)

5. You should see a message that got delivered from the source account EventBridge rule:

   ![Image](/assets/6-1-sqs-poll-for-messages-delivery.png)

6. You can also see messages details by clicking on it:

   ![Image](/assets/6-2-sqs-poll-for-messages-delivery-details.png)

## Testing cross-account Lambda delivery

1. Log-in into destination account and open [Lambda console](https://console.aws.amazon.com/lambda/home) in your browser.
2. Click on the **destination-eb-stack-TargetLambdaFunction-\*** Lambda function name:

   ![Image](/assets/9-lambda-function.png)

3. Click on **Monitor** tab and click on **View CloudWatch logs** button:

   ![Image](/assets/9-1-lambda-function-monitoring.png)

4. Click on CloudWatch log stream name:

   ![Image](/assets/9-2-lambda-function-log-stream.png)

5. Expand **INFO Incoming event** line to see a message payload from the source account:

   ![Image](/assets/9-3-lambda-function-log-stream-details.png)

## Clean-up instructions:

To clean up deployed resourced you can run AWS SAM [delete](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-delete.html) command:

1. Please navigate to **destination-account** folder using your preferred command-line tool and run:

```
sam delete --stack-name destination-eb-stack --profile DESTINATION_ACCOUNT_PROFILE
```

2. Please navigate to **source-account** folder using your preferred command line tool and run:

```
sam delete --stack-name source-eb-stack --profile SOURCE_ACCOUNT_PROFILE
```

In respective folders.

Or you can navigate to [CloudFormation page](https://us-east-1.console.aws.amazon.com/cloudformation/) in your source and destination AWS account and manually delete **source-eb-stack** and **destination-eb-stack** stacks.

## Security

When you run messaging workloads in Production, we strongly recommend to enable encryption for [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/sns-enable-encryption-for-topic.html) and [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-sse-existing-queue.html#sqs-configure-sse-existing-queue-console).

> **Important:** _You must use customer managed KMS key for cross-account access._

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
