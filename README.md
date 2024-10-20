# Project Overview

Users will upload images to a static website hosted on S3 Bucket 1.
The images will be stored in this bucket.
An S3 Event triggers an SQS message.
Lambda reads from SQS, compresses the image, and stores the compressed version in S3 Bucket 2.

## Step 1: Create the First S3 Bucket for Static Website Hosting

### Create the S3 Bucket:
1. Go to the AWS S3 Console.
2. Click Create bucket.
3. Name it something like `image-upload-website`.
4. Select your region.
5. Uncheck the Block all public access option (since this bucket will be public for website access).
6. Click Create bucket.

### Enable Static Website Hosting:
1. Go to the Properties tab of the `image-upload-website` bucket.
2. Scroll down to Static website hosting.
3. Click Edit, select Enable.
4. Set the Index document to `index.html`.
5. Leave Error document blank or specify one if you have a custom error page.
6. Click Save changes.

### Set the Bucket Policy for Public Access:
1. Go to the Permissions tab of the `image-upload-website` bucket.
2. Under Bucket policy, click Edit and paste the following policy to allow public access to your website content:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::image-upload-website/*"
        }
    ]
}
```

3. Replace `image-upload-website` with your actual bucket name.
4. Click Save changes.

## Step 2: Create the Second S3 Bucket for Compressed Images

### Create the Compressed Images Bucket:
1. Go to the S3 console.
2. Click Create bucket.
3. Name it something like `compressed-images-bucket`.
4. Leave the settings as default, making sure Block public access is enabled.
5. Click Create bucket.

## Step 3: Create the SQS Queue for Image Processing

### Create an SQS Queue:
1. Go to the SQS Console.
2. Click Create queue.
3. Choose Standard Queue.
4. Name it something like `image-processing-queue`.
5. Click Create Queue.

## Step 4: Create a Lambda Function to Compress Images

### Create the Lambda Function:
1. Go to the Lambda Console.
2. Click Create function.
3. Choose Author from scratch.
4. Name it `image-compression-lambda`.
5. Choose Python 3.x as the runtime.
6. Click Create function.

### Add SQS Trigger:
1. In the Function overview section, click Add trigger.
2. Select SQS.
3. Choose the `image-processing-queue` created earlier.
4. Click Add.

### Write the Lambda Function Code:
Scroll down to the Function code section and replace the default code with this Python code:

```python
import json
import boto3
from PIL import Image
from io import BytesIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    for record in event['Records']:
        # Get bucket and object key from event
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']

        # Download the image from S3
        response = s3.get_object(Bucket=bucket, Key=key)
        image_data = response['Body'].read()
        img = Image.open(BytesIO(image_data))

        # Compress the image
        compressed_img = BytesIO()
        img.save(compressed_img, format='JPEG', quality=50)

        # Upload compressed image to the compressed S3 bucket
        compressed_bucket = 'compressed-images-bucket'  # Replace with your bucket name
        s3.put_object(Bucket=compressed_bucket, Key=key, Body=compressed_img.getvalue())

    return {
        'statusCode': 200,
        'body': json.dumps('Image processed and uploaded successfully')
    }
```

Replace `compressed-images-bucket` with your actual bucket name where compressed images will be stored.

### Add Permissions to Lambda:
1. In the Configuration tab, click Permissions.
2. Click the Execution Role to view the role in the IAM Console.
3. Attach the following managed policies to the role:
    - AmazonS3FullAccess (or create a custom policy with minimal permissions for reading/writing to both S3 buckets).
    - AWSLambdaSQSQueueExecutionRole.

## Step 5: Create the Static Website (index.html) for Image Upload

### Create the HTML File (index.html):
Create a file called `index.html` with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Image Upload</title>
</head>
<body>
    <h1>Upload an Image</h1>
    <input type="file" id="imageFile" accept="image/*">
    <button onclick="uploadImage()">Upload Image</button>

    <p id="message"></p>

    <script>
        function uploadImage() {
            var file = document.getElementById('imageFile').files[0];
            if (!file) {
                alert('Please choose a file to upload');
                return;
            }

            var formData = new FormData();
            formData.append('file', file);

            fetch('https://image-upload-website.s3.amazonaws.com/', {  // Replace with your bucket's upload endpoint
                method: 'POST',
                body: formData
            })
            .then(response => {
                if (response.ok) {
                    document.getElementById('message').innerText = 'Image uploaded successfully!';
                } else {
                    document.getElementById('message').innerText = 'Image upload failed!';
                }
            })
            .catch(error => console.error('Error:', error));
        }
    </script>
</body>
</html>
```

Replace the fetch URL with the correct pre-signed URL generation logic or API for direct uploads. If using pre-signed URLs, you can generate them using AWS SDK in Python, Node.js, etc.

### Upload the index.html File to Your S3 Bucket:
1. Go back to the `image-upload-website` S3 bucket.
2. Click Upload.
3. Upload the `index.html` file you just created.
4. Click Upload.

## Step 6: Configure S3 Event Notifications to Trigger the SQS Queue

### Set Up S3 Event Notification for the Upload Bucket:
1. Go to the `image-upload-website` bucket.
2. Navigate to Properties > Event Notifications.
3. Click Create event notification.
4. Set Event name to something like `image-upload-event`.
5. For Event types, choose All object create events.
6. For Destination, select SQS Queue and choose the `image-processing-queue`.
7. Click Save.

## Step 7: Test the Workflow

### Access Your Website:
1. Go to the Properties tab of the `image-upload-website` bucket.
2. Copy the Static website hosting URL and open it in a browser. It should load your upload form.

### Upload an Image:
1. Use the form to upload an image.
2. The image will be uploaded to the `image-upload-website` S3 bucket.

### Verify Compression and Workflow:
1. The S3 event triggers a message to the SQS queue.
2. Lambda will process the message, compress the image, and store it in the `compressed-images-bucket`.
3. Check the `compressed-images-bucket` for the compressed version of the image.

## Final Notes
This static website allows users to upload images to S3 directly.
After the image is uploaded, Lambda will automatically compress the image and store it in a different S3 bucket.
The Lambda function, S3 event, SQS, and permissions are set up to ensure the entire workflow is seamless.
