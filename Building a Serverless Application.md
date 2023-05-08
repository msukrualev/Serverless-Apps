# Building a Serverless Application Using Step Functions, API Gateway, Lambda and S3

# Introduction
In this hands-on lab, you will create a fully working serverless reminder application using S3, Lambda, API Gateway, Step Functions, Simple Email Service, and Simple Notification Service. By the end of the lab, you will feel more comfortable architecting and implementing serverless solutions within AWS.

# Solution
Note: Make sure you're in the N. Virginia (us-east-1) region throughout the lab.

Create the Lambda Functions
- In the AWS Management Console, navigate to Lambda.
- In the AWS Lambda console, click Create function.
- Leave the Author from scratch option selected, and then set the following values:

Function name: email
Runtime: Python 3.8

Expand Change default execution role by clicking the arrow next to it.
- Below Execution role, select Use an existing role.
- Go to IAM and create a policy (LambdaLogsSESStatesSNSPolicy) and a role (LambdaRuntimeRole) for Lambda

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "ses:*",
                "states:*",
                "sns:*"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

- Return to Lambda, click refresh then click in the empty drop-down menu that appears below Existing role.
- Select LambdaRuntimeRole from the drop-down menu.
- Click Create function.

- Scroll down to Code source.

- Delete all of the provided code. Copy the code below and paste into lambda_function.py.

```python
import boto3

VERIFIED_EMAIL = 'YOUR_SES_VERIFIED_EMAIL'

ses = boto3.client('ses')

def lambda_handler(event, context):
    ses.send_email(
        Source=VERIFIED_EMAIL,
        Destination={
            'ToAddresses': [event['email']]  # Also a verified email
        },
        Message={
            'Subject': {'Data': 'A reminder from your reminder service!'},
            'Body': {'Text': {'Data': event['message']}}
        }
    )
    return 'Success!'
```


Verify an Email Address in Simple Email Service (SES)

- Open Simple Email Service in a new tab by typing Simple Email Service into the search bar at the top of the screen. 
- In the Amazon Simple Email Service (SES) console, click Create identity.
- On the Create identity page, under Identity type, select Email address.
- Under Email address, enter your personal email address (or an email address you created specifically for this hands-on lab).
- Scroll down and click Create identity.

- In a new browser tab or email client, navigate to your email, open the Amazon SES verification email, and click the provided link.
Note: Check your spam mail if you do not see it in your inbox.
- You should see a Congratulations! page that confirms you successfully verified your email address.

- Return to the AWS SES console tab.
- You should now see Verified under Identity status. (Note: You may need to refresh the page.)

Finish the email Lambda Function Setup
- Return to the Lambda console in your first tab.
- Within the lambda_function code, on line 3, delete the YOUR_SES_VERIFIED_EMAIL placeholder, and type in the email you just used, making sure to leave the single quotes around it.
- Click Deploy in the Code source menu bar above the code.
Your changes have now been deployed.

- Scroll up on the same page to the Function Overview section, and copy the Function ARN by clicking the copy icon next to it, and paste it into a text file for later use in the lab.

Create the sms Lambda Function
- In the navigation line at the top of the page, click Lambda to return to the main Lambda console.
- Click Create function.
- Leave the Author from scratch option selected, and then set the following values:

Function name: sms
Runtime: Python 3.8

- Expand Change default execution role by clicking the arrow next to it.
- Below Execution role, select Use an existing role.
- Once you select that, click into the empty drop-down menu that appears below Existing role.
- Select LambdaRuntimeRole from the drop-down menu.
- Click Create function at the bottom of the page.

- Scroll down to Code source code.
- Delete all of the provided code.
- Copy all of the code and paste into lambda_function.py.

```python
import boto3

sns = boto3.client('sns')

def lambda_handler(event, context):
    sns.publish(PhoneNumber=event['phone'], Message=event['message'])
    return 'Success!'
```

- Click Deploy. Keep this tab open

Create the api_handler Lambda Function
- Scroll up to the navigation line at the top of the page, click Lambda to return to the main Lambda console.
- Click Create function.
- Leave the Author from scratch option selected, and then set the following values:

Function name: api_handler
Runtime: Python 3.8

Expand Change default execution role by clicking the arrow next to it.
- Below Execution role, select Use an existing role.
- Once you select that, click into the empty drop-down menu that appears below Existing role.
- Select LambdaRuntimeRole from the drop-down menu.
- Click Create function at the bottom of the page.

- Scroll down to Code source.
- Delete all of the provided code.
- Copy code below and paste into lambda_function.py.

```python
import boto3
import json
import os
import decimal

SFN_ARN = 'STEP_FUNCTION_ARN'

sfn = boto3.client('stepfunctions')

def lambda_handler(event, context):
    print('EVENT:')
    print(event)
    data = json.loads(event['body'])
    data['waitSeconds'] = int(data['waitSeconds'])
    
    # Validation Checks
    checks = []
    checks.append('waitSeconds' in data)
    checks.append(type(data['waitSeconds']) == int)
    checks.append('preference' in data)
    checks.append('message' in data)
    if data.get('preference') == 'sms':
        checks.append('phone' in data)
    if data.get('preference') == 'email':
        checks.append('email' in data)

    # Check for any errors in validation checks
    if False in checks:
        response = {
            "statusCode": 400,
            "headers": {"Access-Control-Allow-Origin":"*"},
            "body": json.dumps(
                {
                    "Status": "Success", 
                    "Reason": "Input failed validation"
                },
                cls=DecimalEncoder
            )
        }
    # If none, run the state machine and return a 200 code saying this is fine :)
    else: 
        sfn.start_execution(
            stateMachineArn=SFN_ARN,
            input=json.dumps(data, cls=DecimalEncoder)
        )
        response = {
            "statusCode": 200,
            "headers": {"Access-Control-Allow-Origin":"*"},
            "body": json.dumps(
                {"Status": "Success"},
                cls=DecimalEncoder
            )
        }
    return response

# This is a workaround for: http://bugs.python.org/issue16535
class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, decimal.Decimal):
            return int(obj)
        return super(DecimalEncoder, self).default(obj)

```

- Click Deploy. Keep this tab open.

Create a Step Function State Machine
- Open the AWS Step Functions console in a new tab by typing Step Functions into the search bar at the top of the screen.

- In the AWS Step Functions console tab, click Get started.

- In the line under the Review Hello World example, click "here" at the end of the sentence to create your own step function.

Note: If prompted with a Leave page? warning pop-up window, click Leave, and then click "here" on the line under Review Hello World example again.

- On the Create state machine page, under Choose authoring method, select Write your workflow in code.
- Under Type, leave Standard selected.
- Scroll down to Definition and delete all of the provided code.
- Copy all of the code and paste the code under Definition.

```json
{
  "Comment": "An example of the Amazon States Language using a choice state.",
  "StartAt": "SendReminder",
  "States": {
    "SendReminder": {
      "Type": "Wait",
      "SecondsPath": "$.waitSeconds",
      "Next": "ChoiceState"
    },
    "ChoiceState": {
      "Type" : "Choice",
      "Choices": [
        {
          "Variable": "$.preference",
          "StringEquals": "email",
          "Next": "EmailReminder"
        },
        {
          "Variable": "$.preference",
          "StringEquals": "sms",
          "Next": "TextReminder"
        },
        {
          "Variable": "$.preference",
          "StringEquals": "both",
          "Next": "BothReminders"
        }
      ],
      "Default": "DefaultState"
    },

    "EmailReminder": {
      "Type" : "Task",
      "Resource": "EMAIL_REMINDER_ARN",
      "Next": "NextState"
    },

    "TextReminder": {
      "Type" : "Task",
      "Resource": "TEXT_REMINDER_ARN",
      "Next": "NextState"
    },
    
    "BothReminders": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "EmailReminderPar",
          "States": {
            "EmailReminderPar": {
              "Type" : "Task",
              "Resource": "EMAIL_REMINDER_ARN",
              "End": true
            }
          }
        },
        {
          "StartAt": "TextReminderPar",
          "States": {
            "TextReminderPar": {
              "Type" : "Task",
              "Resource": "TEXT_REMINDER_ARN",
              "End": true
            }
          }
        }
      ],
      "Next": "NextState"
    },
    
    "DefaultState": {
      "Type": "Fail",
      "Error": "DefaultStateError",
      "Cause": "No Matches!"
    },

    "NextState": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

- On lines 34 and 52, replace the EMAIL_REMINDER_ARN placeholder value with the copied ARN for email Lambda function that you should have copied over onto a text file earlier, being sure to leave the double quotes around the ARN.

- On lines 40 and 62, replace the TEXT_REMINDER_ARN placeholder value with the copied ARN for sms Lambda function.

You should now see the updated function diagram to the right of the code.

- Click Next at the bottom of the screen.
- On the Specify details page, you can leave the default name, and under Permissions, select Choose an existing role.
- Go to IAM and create a policy (step-lambda-policy) and a role (step-lambda-role) for Step Functions using the code below. Do not forget to choose use case Step Functions.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

- Return to Step Functions then select RoleForStepFunction under Existing roles.
- Scroll down, and click Create state machine.
- Your state machine should now be created.

- On the MyStateMachine page, under Details, copy the ARN to your clipboard.
- Return to the Lambda console.
- In the navigation line at the top of the page, click Functions.
- Under Functions, select the api_handler function.
- Scroll down to Code source and view the lambda_function.py code.
- On line 6, replace the STEP_FUNCTION_ARN placeholder with the ARN you just copied.
- Click Deploy.

Create the API Gateway
- Open the Amazon API Gateway console in a new tab by typing API Gateway into the search bar at the top of the screen.
- In your Amazon API Gateway console tab, scroll down to REST API (the one that does not say Private), and then select Build.
- In the Create your first API pop-up window, click OK.
- Under Choose the protocol, leave REST selected.
- Under Create new API, select New API.
- Under Settings, set the following values:

API name: reminders
Leave Description blank.
Endpoint Type: Regional
- Click Create API at the bottom of the screen.

You should now be in your API Gateway, and you will see that you don't have any methods defined for the resource.

- Click on the / under Resources.
- Click Actions > Create Resource.
- On the New Child Resource page, next to Resource Name, enter reminders, which will also auto-populate /reminders as the Resource Path.
- Select the checkbox next to Enable API Gateway CORS.
- Click Create Resource at the bottom of the page.

- Then, click /reminders, and click Actions > Create Method.
- In the dropdown that appears under /reminders, select POST and click the adjacent checkmark icon.
- Under /reminders - POST - Setup, set the following values:

Integration type: Lambda Function
Use Lambda Proxy integration: Select the checkbox.
Lambda Region: us-east-1
Lambda Function: Start typing, and then select, api_handler.
- Click Save at the bottom of the page

- In the Add Permission to Lambda Function pop-up window, click OK.

- Click Actions > Deploy API.

In the Deploy API pop-up window, set the following values:

Deployment stage: [New Stage]
Stage name: prod
- Click Deploy.

Note: You may ignore any Web Application Firewall (WAF) permissions warning messages if received after deployment.

- At the top of the page, under prod Stage Editor, copy the Invoke URL and paste it into a text file for later use in the lab..

Create and Test the Static S3 Website
- In the static web site folder open the formlogic.js file in a text editor.
- On line 5 of formlogic.js, delete the UPDATETOYOURINVOKEURLENDPOINT placeholder. Paste in the Invoke URL you previously copied from API Gateway, being sure to keep /reminders on the end of the string.
- Save the local file.

- Open the Amazon S3 console in a new tab by typing S3 into the search bar at the top of the screen.
- In the Amazon S3 console, click Create bucket.
- On the Create bucket page, under Bucket name, enter a globally unique bucket name.
- Uncheck Block all public access.
- Under Block all public access, select I acknowledge that the current settings might result in this bucket and the objects within becoming public.
- Leave the other defaults, scroll down, and click Create bucket.

- Select the new bucket under Buckets to open it.
- On the bucket page, click Upload.
- On the Upload page, click Add files and select all of your local files from the static_website folder, and click Open. Or, drag all of your local files from the static_website folder into the Drag and drop box under Upload.

cat.png
error.html
formlogic.js
IMG_0991.jpg
index.html
main.css

- Once you see the uploads have succeeded, click Close in the top right-hand corner of the page.
- Use Permissions, Bucket Policy to edit bucket policy to enable public access.
- Click on the Properties tab under the bucket name.
- Scroll down to Static website hosting, and click Edit.
- On the Edit static website hosting page, under Static website hosting, select Enable.

Set the following values:

Index document: index.html
Error document: error.html
- Click Save changes at the bottom of the page.

- Scroll down again to Static website hosting, and click the URL below Bucket website endpoint to access the webpage.

- Once you are on the static website, to test the service's functionality, set the following values:

Seconds to wait: 1
Message: Hello!
someone@something.com: Your personal email address (This has to be the same one you verified with Simple Email Service earlier.)
Under Reminder Type, select email.

- You should see Looks ok. But check the result below! above the Required sections, and you should see {"Status":"Success"} at the bottom of the page.

- In a new browser tab or email client, navigate to your email. You should now see a reminder email from the service.
Note: Check your spam folder if you do not see it in your inbox.

- Navigate to your AWS Step Functions console tab.
- In the Executions section, click on the refresh icon.
- You should see at least one execution displayed.
- Click on one of the executions.
- Scroll down to Graph inspector to view the event's visual workflow.

Conclusion
Congratulations — you've completed this hands-on lab!
