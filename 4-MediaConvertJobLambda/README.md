# Automating  MediaConvert Jobs with Lambda

In this phase, you'll use Amazon S3 events and Lambda to automatically trigger AWS Elemental MediaConvert jobs. The ability to watch S3 folders and take action on incoming items is a useful automation technique that enables the creation of fully unattended workflows. In our case, the user will place videos in a folder on AWS S3 which will trigger an ingest workflow to convert the video.  We'll use the job definition from the previous phase as the basis for our automated job.  

![Serverless transcoding architecture](../images/lamda%20workflow.png)

You'll implement a Lambda function that will be invoked each time a user uploads a video.  The lambda will send the video to the MediaConvert service to produce an Apple HLS adaptive bitrate stream for playout on multiple sized devices and varying bandwidths.


Converted outputs will be saved in the destination S3 MediaBucket created in earlier in the phase.

## Implementation Instructions
### [1. Create an IAM Role for Your Lambda function](#lambda-role)

#### Background

Every Lambda function has an IAM role associated with it. This role defines what other AWS services the function is allowed to interact with. For the purposes of this workshop, you'll need to create an IAM role that grants your Lambda function permission to interact with the MediaConvert service.

### High-Level Instructions

Use the IAM console to create a new role. Name it `VODLambdaRole` and select AWS Lambda for the role type. 

Attach the managed policy called `AWSLambdaBasicExecutionRole`, `AmazonS3FullAccess` to this role to grant the necessary CloudWatch Logs permissions. 

Use inline policies to grant permissions to other resources needed for the lambda to execute.

### Step-by-step instructions

1. From the AWS Management Console, click on **Services** and then select **IAM** in the Security, Identity & Compliance section.

1. Select **Roles** in the left navigation bar and then choose **Create role**.

1. Select **AWS Service** and **Lambda** for the role type, then click on the **Next:Permissions** button.

    **Note:** Selecting a role type automatically creates a trust policy for your role that allows AWS services to assume this role on your behalf. If you were creating this role using the CLI, AWS CloudFormation or another mechanism, you would specify a trust policy directly.

1. Begin typing `AWSLambdaBasicExecutionRole`in the **Filter** text box and check the box next to that role.

1. Delete what you entered in the **Filter** text box, and this time search for **AmazonS3FullAccess**. Check the box next to this role. 

1. Choose **Next:Tags**.

1. Choose **Next:Review**.

1. Enter `VODLambdaRole` for the **Role name**.

1. Choose **Create role**.

1. Type `VODLambdaRole` into the filter box on the Roles page and choose the role you just created.

1. On the Permissions tab, click on the **Add Inline Policy** link and choose the **JSON** tab.

1. Copy and paste the following JSON in the **Policy Document Box**.  You will need to edit this policy in the next step to fill in the resources for your application.

```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "Logging"
        },
        {
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "<ARN for MediaConvertRole>"
            ],
            "Effect": "Allow",
            "Sid": "PassRole"
        },
        {
            "Action": [
                "mediaconvert:*"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow",
            "Sid": "MediaConvertService"
        }
    ]
}
```
1. Replace <ARN for vod-MediaConvertRole> tag in the policy with the ARN for the vod-MediaConvertRole you created earlier.
2. Click on the **Review Policy** button.

1. Enter `VODLambdaPolicy` in the **Policy Name** box.

1. Click on the **Create Policy** button.

### [ 2. Create a Lambda Function for converting videos]()

#### Background

AWS Lambda will run your code in response to events such as a putObject into S3 or an HTTP request. In this step you'll build the core function that will process videos using the MediaConvert python SDK. The lambda function will respond to putObject events in your S3 source bucket.  Whenever a video file is added, the lambda will start a MediaConvert job.

This lambda will submit a job to MediaConvert, but it won't wait for the job to complete.

### High-Level Instructions

Use the AWS Lambda console to create a new Lambda function called `VODLambdaConvert` that will process the API requests. Use the provided [lambda_convert.py](lambda_convert.py) example implementation for your function code. 

Make sure to configure your function to use the `VODLambdaRole` IAM role you created in the previous section.

#### Step-by-step instructions 

1. Choose **Services** then select **Lambda** in the Compute section.

1. Choose **Create function**.

1. Choose the **Author from scratch** button.

1. On the **Author from Scratch** panel, enter `LambdaConvert` in the **Function name** field.
2. Select **Python 3.9** for the **Runtime**.

1. Expand the **Choose or create an execution role**.

1. Select **Use an existing role**. 

1. Select `VODLambdaRole` from the **Existing Role** dropdown.

1. Click on **Create function**.

    ![Create Lambda convert function screenshot](../images/lambda%20_create.png)

1. On the Configuration tab of the LambdaConvert page, in the  **function code** panel:  

    1. upload the zip file given in this repo folder.To view click [here](./lambda.zip). 
    
        Note that this zip file is simply the [lambda_convert.py](lambda_convert.py) script and the [job.json](job.json) file provided in this repo that you could zip up yourself, if desired.

    1. Enter `lambda_convert.handler` for the **Handler** field.

        ![Lambda function code screenshot](../images/lambda_handler.png)

1. On the configuration tab select **Environment Variables** panel of the LambdaConvert page, enter the following keys and values:

    * DestinationBucket = destination bucket(or whatever you named your bucket in phase 1)
    * MediaConvertRole = arn:aws:iam::ACCOUNT NUMBER:role/vod-MediaConvertRole (ARN of MediaConvert role)
    * Application = lambda

    ![Lambda function code screenshot](../images/lambda_env_var.png)

1.  On the configuration tab select **General configuration** panel of the LambdaConvert page, set **Timeout** to 2 minutes.

1. Scroll back to the bottom of the page and click on the **Save** button.

### create a test event
1. switch to s3 in a new tab,then select the source bucket (watch folder bucket)

1. select upload , then select add files and choose the video file that you want to convert.

1. As soon as the file uploaded in your watch folder bucket you will receive a s3 bucket event via SNS email notification.

1.copy the json event notification.you will need this for testing lamda.

1.sample event structure on uploading [waterfall.mp4](../3-S3andSNS/waterfall.mp4)
```JSON
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "ap-south-1",
      "eventTime": "2022-11-29T05:49:26.809Z",
      "eventName": "ObjectCreated:CompleteMultipartUpload",
      "userIdentity": {
        "principalId": "A215A9FN6IYIR1"
      },
      "requestParameters": {
        "sourceIPAddress": "106.197.78.77"
      },
      "responseElements": {
        "x-amz-request-id": "CFW34C1BJGJ8X7CV",
        "x-amz-id-2": "nS9pUwmzfdM0npMAiV3MIW0Brh+HpyvivnGd20t+ra/up/p3cUl35a1nJt91DkTVKE2GrosR61Fgi4vfcNwPbdy2MIHTProm"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "s3 event",
        "bucket": {
          "name": "database24",
          "ownerIdentity": {
            "principalId": "A215A9FN6IYIR1"
          },
          "arn": "arn:aws:s3:::database24"
        },
        "object": {
          "key": "waterfall.mp4",
          "size": 32407849,
          "eTag": "ae049aede2c55666f7d39ff06368bb30-2",
          "sequencer": "0063859D5B81D5AB9F"
        }
      }
    }
  ]
}
  


```



### Test the lambda

1. From the main edit screen for your function, select **Test**.

    ![Configure test event](../images/lambda_test.png)

1. Enter `ConvertTest` in the **Event name** box.


1. In **Event JSON** copy and paste the s3 file creation event in json format that you received via email using SNS and then click format JSON.

1. Click on **Create** button. 
1. Then back on the main page, click on **Test** button.
1. Verify that the execution succeeded and that the function result looks like the following:
    ```JSON
    {
    "body": "{}",
    "headers": {
        "Access-Control-Allow-Origin": "*",
        "Content-Type": "application/json"
    },
    "statusCode": 200
    }
    ```

### 6. Create a S3 PutItem Event Trigger for your Convert lambda

#### Background

In the previous step, you built a lambda function that will convert a video in response to an S3 PutItem event.  Now it's time to hook up the Lambda trigger to the watchfolder S3 bucket.

#### High-Level Instructions

Use the AWS Lambda console to add a putItem trigger from the `watchfolder source` S3 bucket to the `LambdaConvert` lambda function.  

#### Step-by-step instructions

1. In the **Configuration->Designer** panel of the VODLambdaConvert function, click on **Add trigger** button. 
1. Select **S3** from the **Trigger configuration** dropdown.
1. Select `vod-watchfolder-firstname-lastname` or the name you used for the watchfolder bucket you created earlier in this phase for the **Bucket**.
1. Select **PUT** for the **Event type**.

    ![Lambda S3 trigger](../images/lambda_trigger.png)

1. Leave the rest of the settings as the default and click the **Add** button.

    ![Lambda trigger configuration screenshot](../images/lambda_config.png)


### Test the watchfolder automation

You can use your own video or use the test.mp4 video included in this folder to test the workflow. 

In the next phase, we will setup automated monitoring of jobs created using the watchfolder workflow.  Until then, you can monitor the the MediaConvert console.  

1. Open the S3 console overview page for the watchfolder S3 bucket you created earlier.
1. Select **Upload** and then choose the file [test.mp4](test.mp4) from the directory for this repo phase on your computer.
1. Note the time that the upload completed.
1. Open the MediaConvert jobs page and find a job for the input 'test.mp4' that was started near the time your upload completed.  

    ![Lambda trigger configuration screenshot](../images/lamda_mediaconvert.png)

1. Navigate to the HLS output of destination s3 bucket and play the test video by copying the Object URL of test.mp4 and paste it in HLS player.

You can play the HLS using:
* Safari browser by clicking on the **Link** for the object.
* **JW Player Stream Tester** - by copying the link for the object and inputing it to the player.  https://developer.jwplayer.com/tools/stream-tester/ 


    ![test video](../images/play-test-video.png)



