# Hackathon Signup Website

This project sets up a hackathon signup website using AWS services. Users can enter their email addresses, which are then processed by an AWS Lambda function and stored in an S3 bucket. The frontend is hosted on S3 as a static website.

## Project Directory Structure

```

HACK-CHATT/
├── backend/
│ ├── lambda.py
│ ├── lambda_function.zip # This will be created by zipping lambda.py
├── frontend/
│ ├── index.html
│ └── assets/
│ └── (any additional front-end assets like CSS, JS, images)
├── terraform/
│ ├── main.tf
│ ├── providers.tf
│ ├── variables.tf
├── .gitignore
└── README.md

```

## Backend

### `lambda.py`

This Lambda function handles the email submissions and stores them in an S3 bucket.

```python
import json
import boto3
import os

s3 = boto3.client('s3')
bucket_name = os.environ['EMAIL_STORAGE_BUCKET']

def lambda_handler(event, context):
    body = json.loads(event['body'])
    email = body.get('email')
    if email:
        try:
            # Read existing emails from S3
            existing_emails = []
            try:
                response = s3.get_object(Bucket=bucket_name, Key='emails.json')
                existing_emails = json.loads(response['Body'].read().decode('utf-8'))
            except s3.exceptions.NoSuchKey:
                # If the file doesn't exist, initialize with an empty list
                existing_emails = []

            # Add new email to the list
            existing_emails.append(email)

            # Write updated list back to S3
            s3.put_object(Bucket=bucket_name, Key='emails.json', Body=json.dumps(existing_emails))

            return {
                'statusCode': 200,
                'body': json.dumps({'message': 'Signed up successfully!'})
            }
        except Exception as e:
            return {
                'statusCode': 500,
                'body': json.dumps({'message': 'Error signing up.'})
            }
    return {
        'statusCode': 400,
        'body': json.dumps({'message': 'Invalid request.'})
    }
```

### Create Deployment Package

Navigate to the `backend` directory and create a zip file of the Lambda function:

```sh
cd backend
zip lambda_function.zip lambda.py
```

## Frontend

### `index.html`

This HTML file contains the form for users to enter their email addresses.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Hackathon Signup</title>
  </head>
  <body>
    <h1>Sign Up for the Hackathon</h1>
    <form id="emailForm">
      <label for="email">Email:</label>
      <input type="email" id="email" name="email" required />
      <button type="submit">Sign Up</button>
    </form>
    <script>
      document
        .getElementById("emailForm")
        .addEventListener("submit", async function (event) {
          event.preventDefault();
          const email = document.getElementById("email").value;
          try {
            const response = await fetch(
              "YOUR_API_GATEWAY_ENDPOINT/subscribe",
              {
                method: "POST",
                headers: {
                  "Content-Type": "application/json",
                },
                body: JSON.stringify({ email }),
              }
            );
            if (response.ok) {
              alert("Signed up successfully!");
            } else {
              alert("Failed to sign up.");
            }
          } catch (error) {
            console.error("Error:", error);
            alert("An error occurred.");
          }
        });
    </script>
  </body>
</html>
```

## Terraform Configuration

### `variables.tf`

```hcl
variable "region" {
  default = "us-east-1"
}

variable "lambda_function_name" {
  default = "HackathonEmailSignup"
}

variable "website_bucket_name" {
  default = "hackathon-website-bucket"
}

variable "email_storage_bucket_name" {
  default = "hackathon-email-storage"
}

variable "lambda_s3_key" {
  default = "lambda_function.zip"
}
```

### `main.tf`

```hcl
provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "website_bucket" {
  bucket = var.website_bucket_name

  website {
    index_document = "index.html"
  }
}

resource "aws_s3_bucket_website_configuration" "website_config" {
  bucket = aws_s3_bucket.website_bucket.id

  index_document {
    suffix = "index.html"
  }
}

resource "aws_s3_bucket_ownership_controls" "website_bucket_ownership" {
  bucket = aws_s3_bucket.website_bucket.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_public_access_block" "website_bucket_public_access" {
  bucket = aws_s3_bucket.website_bucket.id

  block_public_acls   = true
  block_public_policy = true
  ignore_public_acls  = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket" "email_storage_bucket" {
  bucket = var.email_storage_bucket_name
}

resource "aws_s3_bucket_ownership_controls" "email_storage_bucket_ownership" {
  bucket = aws_s3_bucket.email_storage_bucket.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_public_access_block" "email_storage_bucket_public_access" {
  bucket = aws_s3_bucket.email_storage_bucket.id

  block_public_acls   = true
  block_public_policy = true
  ignore_public_acls  = true
  restrict_public_buckets = true
}

resource "aws_iam_role" "lambda_role" {
  name = "lambda_execution_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "lambda.amazonaws.com"
        },
      },
    ],
  })
}

resource "aws_iam_policy" "lambda_policy" {
  name        = "lambda_s3_policy"
  description = "Policy for Lambda to access S3"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = [
          "s3:PutObject",
          "s3:GetObject"
        ],
        Effect   = "Allow",
        Resource = "arn:aws:s3:::${var.email_storage_bucket_name}/*",
      },
    ],
  })
}

resource "aws_iam_role_policy_attachment" "lambda_policy_attachment" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

resource "aws_lambda_function" "email_signup" {
  function_name = var.lambda_function_name
  role          = aws_iam_role.lambda_role.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.8"
  filename      = "../backend/lambda_function.zip"

  source_code_hash = filebase64sha256("../backend/lambda_function.zip")

  environment {
    variables = {
      EMAIL_STORAGE_BUCKET = var.email_storage_bucket_name
    }
  }
}

resource "aws_api_gateway_rest_api" "email_signup_api" {
  name = "EmailSignupAPI"
}

resource "aws_api_gateway_resource" "signup_resource" {
  rest_api_id = aws_api_gateway_rest_api.email_signup_api.id
  parent_id   = aws_api_gateway_rest_api.email_signup_api.root_resource_id
  path_part   = "subscribe"
}

resource "aws_api_gateway_method" "post_method" {
  rest_api_id   = aws_api_gateway_rest_api.email_signup_api.id
  resource_id   = aws_api_gateway_resource.signup_resource.id
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "lambda_integration" {
  rest_api_id = aws_api_gateway_rest_api.email_signup_api.id
  resource_id = aws_api_gateway_resource.signup_resource.id
  http_method = aws_api_gateway_method.post_method.http_method
  type        = "AWS_PROXY"
  integration_http_method = "POST"
  uri         = aws_lambda_function.email_signup.invoke_arn
}

resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.email_signup.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.email_signup_api.execution_arn}/*/*"
}

resource "aws_api_gateway_deployment" "api_deployment" {
  depends_on = [
    aws_api_gateway_integration.lambda_integration,
  ]

  rest_api_id = aws_api_gateway_rest_api.email_signup_api.id
  stage_name  = "prod"
}
```

## Deployment Steps

1. **Prepare the Lambda Function Deployment Package**:

   - Navigate to your `backend` directory and create a zip file of your Lambda function:

     ```sh
     cd backend
     zip lambda_function.zip lambda.py
     ```

2. **Initialize and Apply Terraform Configuration**:

   - Navigate to the `terraform` directory:

     ```sh
     cd terraform
     terraform init
     terraform apply
     ```

3. **Upload `index.html` to S3**:

   - Navigate to your `frontend` directory and upload `index.html` to your website bucket:

     ```sh
     aws s3 cp index.html s3://hackathon-website-bucket/
     ```

   - repeat for styles.css, success.html, and privacy-policy.html (This needs to be automated but for now, do it manually)
   - for the assets folder create an assets folder in the bucket and then upload the assets to the assets folder like this:

     ```sh
      `aws s3 cp assets s3://hackathon-website-bucket/assets --recursive`
     ```

4. **Set Bucket Policies**:

   - Use the AWS CLI to set the bucket policy for the email storage bucket:

     ```sh
     aws s3api put-bucket-policy --bucket hackathon-email-storage --policy '{
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": {
             "Service": "lambda.amazonaws.com"
           },
           "Action": "s3:PutObject",
           "Resource": "arn:aws:s3:::hackathon-email-storage/*"
         },
         {
           "Effect": "Allow",
           "Principal": {
             "AWS": "arn:aws:iam::your-account-id:role/organizers-role"
           },
           "Action": "s3:GetObject",
           "Resource": "arn:aws:s3:::hackathon-email-storage/*"
         }
       ]
     }'
     ```

   - Use the AWS CLI to set the bucket policy for the website bucket:

     ```sh
     aws s3api put-bucket-policy --bucket hackathon-website-bucket --policy '{
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "PublicReadGetObject",
           "Effect": "Allow",
           "Principal": "*",
           "Action": "s3:GetObject",
           "Resource": "arn:aws:s3:::hackathon-website-bucket/*"
         }
       ]
     }'
     ```

5. **Update Frontend with API Gateway Endpoint**:

   - Retrieve your API Gateway endpoint using the AWS CLI:

     ```sh
     aws apigateway get-rest-apis
     ```

   - Replace `YOUR_API_GATEWAY_ENDPOINT` in `index.html` with your actual API Gateway endpoint URL.

6. **Testing the Lambda Function**:

   - In the AWS Lambda console, create a test event with the following JSON payload:

     ```json
     {
       "body": "{\"email\":\"test@example.com\"}"
     }
     ```

   - Execute the test and verify that the email is stored in the S3 bucket.

## Conclusion

Following these steps will deploy your hackathon signup website using AWS services. Users can sign up with their email addresses, which will be processed by a Lambda function and stored securely in an S3 bucket. The website is hosted on S3 for a simple, scalable, and cost-effective solution.

---
