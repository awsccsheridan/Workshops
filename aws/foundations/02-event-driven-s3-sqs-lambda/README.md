# 🚀 AWS Foundations: Event-Driven Pipelines with S3, SQS, and Lambda

Hosted by **Preyas Patel** <br>
Part of the **Cloud & DevOps Workshop Series**
Recording : https://sheridanc.sharepoint.com/:v:/s/FAST-AWSCloudClub/IQCRnB3pUhLUQZpm8Djd5p9aATOFnMKhEh9qkTD85-1n8_8?e=IldSdm

🕒 Duration: 2–3 Hours <br>
📈 Difficulty: Intermediate <br>
💰 Estimated Cost: ~$0.50–$1.00 (Free Tier eligible) <br>
🌎 Recommended Region: `us-east-1` <br>

---

## 🎥 Workshop Resources

| Resource               | Link                                           |
| ---------------------- | ---------------------------------------------- |
| 📄 Workshop Guide      | This README                                    |
| 🧩 Lambda Code         | Included below                                 |
| 🧪 Load Testing Script | Included below                                 |

---

# 📌 Overview

In this workshop, we build a **real-world event-driven serverless image processing pipeline** using AWS managed services.

When an image is uploaded to Amazon S3:

1. S3 emits an event notification
2. The event is delivered to Amazon SQS
3. SQS buffers and triggers AWS Lambda
4. Lambda resizes the image using Pillow
5. The processed image is stored in a separate S3 bucket
6. Logs and metrics are captured in CloudWatch
7. Failed messages are routed to a Dead Letter Queue (DLQ)

This architecture reflects production-grade serverless patterns used in modern cloud systems.

---

# 🏗 Architecture Overview

## 🔄 Data Flow

1. Upload image → **S3 (uploads bucket)**
2. S3 event → **SQS queue**
3. SQS triggers → **Lambda function**
4. Lambda processes image
5. Output stored → **Processed S3 bucket**
6. Logs streamed → **CloudWatch**
7. Failures moved → **Dead Letter Queue**

---

## 🧩 Services Used

| Service       | Role                          |
| ------------- | ----------------------------- |
| Amazon S3     | Object storage & event source |
| Amazon SQS    | Decoupling and reliability    |
| AWS Lambda    | Serverless compute            |
| Lambda Layers | Dependency management         |
| CloudWatch    | Monitoring and logging        |

---

# 🧠 Prerequisites

* AWS account
* Permissions for S3, SQS, Lambda, IAM, CloudWatch
* Basic Python knowledge
* AWS CLI configured (recommended)

---

# 🛠 Module 1 – Foundation Setup

---

## Step 1.1 – Create S3 Buckets

Create two globally unique buckets:

```
workshop-image-uploads-[your-name]
workshop-image-processed-[your-name]
```

### CLI (Optional)

```bash
aws s3 mb s3://workshop-image-uploads-[your-name] --region us-east-1
aws s3 mb s3://workshop-image-processed-[your-name] --region us-east-1
```

Keep:

* Block Public Access: Enabled
* Versioning: Disabled

---

## Step 1.2 – Create SQS Queue + DLQ

### Dead Letter Queue

```
ImageProcessingQueue-DLQ
```

### Main Queue

```
ImageProcessingQueue
```

### Recommended Settings

* Visibility Timeout: 360 seconds
* Receive Wait Time: 20 seconds
* Max Receives: 3
* Dead Letter Queue: Enabled

> ⚠ Visibility timeout must be greater than Lambda timeout.

---

## Step 1.3 – Create IAM Role for Lambda

Role Name:

```
LambdaImageProcessorRole
```

Attach Managed Policies:

* AWSLambdaSQSQueueExecutionRole
* AWSLambdaBasicExecutionRole

Add Inline S3 Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": [
        "arn:aws:s3:::workshop-image-uploads-*/*",
        "arn:aws:s3:::workshop-image-processed-*/*"
      ]
    }
  ]
}
```

---

# 🧠 Module 2 – Lambda Development

---

## Step 2.1 – Add Pillow Lambda Layer

### Public Layer (us-east-1)

```
arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p312-Pillow:9
```

### OR Build Your Own

```bash
mkdir python
pip install Pillow -t python/
zip -r pillow-layer.zip python

aws lambda publish-layer-version \
  --layer-name pillow-layer \
  --zip-file fileb://pillow-layer.zip \
  --compatible-runtimes python3.11
```

---

## Step 2.2 – Create Lambda Function

Configuration:

* Name: `ImageProcessor`
* Runtime: Python 3.11
* Memory: 512 MB
* Timeout: 60 seconds
* Architecture: x86_64
* Role: LambdaImageProcessorRole

Environment Variable:

```
PROCESSED_BUCKET=workshop-image-processed-[your-name]
```

---

## Lambda Code

```python
import json
import boto3
import os
from PIL import Image
from io import BytesIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    processed_count = 0
    failed_count = 0

    for record in event['Records']:
        try:
            message_body = json.loads(record['body'])

            if 'Event' in message_body and message_body['Event'] == 's3:TestEvent':
                continue

            s3_record = message_body['Records'][0]
            bucket_name = s3_record['s3']['bucket']['name']
            object_key = s3_record['s3']['object']['key']

            response = s3.get_object(Bucket=bucket_name, Key=object_key)
            image = Image.open(BytesIO(response['Body'].read()))
            image.thumbnail((800, 600), Image.Resampling.LANCZOS)

            buffer = BytesIO()
            image.save(buffer, format=image.format or 'JPEG')
            buffer.seek(0)

            s3.put_object(
                Bucket=os.environ['PROCESSED_BUCKET'],
                Key=f"processed/{object_key}",
                Body=buffer.getvalue(),
                ContentType=response['ContentType']
            )

            processed_count += 1

        except Exception:
            failed_count += 1
            raise

    return {
        "processed": processed_count,
        "failed": failed_count
    }
```

---

# 🔗 Module 3 – Event Integration

### Grant S3 Permission to Send to SQS

Replace `YOUR-ACCOUNT-ID`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:YOUR-ACCOUNT-ID:ImageProcessingQueue",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::workshop-image-uploads-*"
        }
      }
    }
  ]
}
```

Configure S3 Event:

* Event: ObjectCreated:*
* Destination: SQS
* Queue: ImageProcessingQueue

Connect SQS to Lambda:

* Batch size: 5
* Batch window: 10 seconds

---

# 🧪 Testing & Monitoring

Upload test image:

```bash
aws s3 cp cat.jpg s3://workshop-image-uploads-[your-name]/test/cat.jpg
```

Verify logs:

```
/aws/lambda/ImageProcessor
```

Load test:

```bash
for i in $(seq 1 100); do
  aws s3 cp cat.jpg s3://workshop-image-uploads-[your-name]/load-test/image-$i.jpg &
done
wait
```

---

# 🧹 Cleanup

Delete in this order:

1. Remove Lambda trigger
2. Delete Lambda function
3. Delete SQS queues
4. Empty & delete S3 buckets
5. Delete IAM role

---

# 🎯 Key Takeaways

* Event-driven systems reduce coupling
* SQS ensures reliability and durability
* Lambda scales automatically
* DLQs are critical for production systems
* Observability is essential in distributed architectures

---

# ⭐ Support the Workshop Series

If this workshop helped you:

* Star the repository
* Share with peers
* Attend future sessions

---
