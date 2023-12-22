# AWS Integration with Refuel

This document describes how you can set up an AWS integration with Refuel. This integration allows Refuel to read your data directly from S3 in order to:

- Display it in the Refuel web application
- Perform some necessary processing, including extracting metadata, performing OCR, etc.

In the first case, a pre-signed URL is created for the data that expires after a short time, and in the second case, the data is promptly deleted from the Refuel backend after processing.

## Setup

### Refuel AWS Account ID and External ID

Email [support@refuel.ai](mailto:support@refuel.ai) to receive the Refuel AWS Account ID and external ID that you can use in the later steps.

This will be made available in the UI at a later point.

### Configure CORS

CORS (Cross-origin resource sharing) is a browser mechanism that enables restricted access across domain boundaries that would otherwise be prohibited by default browser restrictions. The CORS header enables Refuel to send a pre-flight request to your cloud storage and enables your cloud storage to explicitly allow requests from Refuel. If the Refuel domains are included in the CORS header, Refuel will be able to request resources from your cloud storage.

When configuring CORS for your cloud storage bucket, you will need to include the following Refuel origin: [https://app.refuel.ai](https://app.refuel.ai).

- In your AWS account, go to your S3 Management Console
- Click the bucket name in the list of buckets.
- Go to the **Permissions** tab.
- In the Cross-origin resource sharing (CORS) section, click Edit.
- Paste the following configuration in the text field.

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET"],
    "AllowedOrigins": ["https://app.refuel.ai"],
    "ExposeHeaders": []
  }
]
```

- Click Save changes.
- For more details on setting up CORS for your AWS S3 bucket, see these [AWS docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enabling-cors-examples.html).

### Create IAM Policy for S3 Access

- Navigate to the IAM Console in AWS, click on Policies in the left navbar, and then hit the **Create Policy** button.
- Click on the JSON tab, copy the policy below (substitute your bucket name, add multiple buckets, restrict to specific keys in a bucket) and then hit Next.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::YourBucketName/*",
        "arn:aws:s3:::YourBucketName"
      ]
    }
  ]
}
```

- Click Next again, give the policy a name, such as **RefuelS3ReadAccessPolicy** and then hit **Create policy**.

### Create IAM Role for S3 Access

- In the IAM Console, click on Roles in the left navbar and click on **Create role**
- In **Trusted entity type**, click on **AWS account** and then click on **Another AWS account**.
- Paste in the Refuel AWS account ID from step 1.
- Click **Require external ID** and again paste in the external ID from step 1, and click Next.
- Search for, and add the policy you just created: **RefuelS3ReadAcccessPolicy** and hit Next.
- Give this role a name, let’s say **RefuelS3ReadAccessRole** and click **Create role**.

### Send Refuel the ARN of the newly created role

Email [support@refuel.ai](mailto:support@refuel.ai) with the role ARN.

At a later point, this will just be done in the Refuel web application.

That’s it – now start sending Refuel your data!

When you send Refuel any pieces of content that sit in S3, all you need to send is the path in S3 ([s3://YourBucketName/path/to/key](s3://YourBucketName/path/to/key)), and Refuel will be able to securely access it.
