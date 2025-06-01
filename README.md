AWS homelab setup
=================
Basic AWS homelab setup with Localstack, awscli-local and terraform-local.

---

## Installations

1. Setup venv
```sh
deactivate  # If there's any python virtual environmnent already activated.
python3 -m venv ls-venv
source ./ls-venv/bin/activate
pip install -r requirements.txt
```

2. Start localstack
```sh
localstack start -d
localstack status services
```

---

## Run the lambda + apigateway example
```sh
cd lambda && zip ../lambda.zip index.js && cd -
tflocal apply -auto-approve
curl http://localhost:4566/restapis/$(awslocal apigateway get-rest-apis --query 'items[0].id' --output text)/dev/_user_request_/hello
```

---

## Explaining main.tf

**Step 1: Set up the AWS “Lego Table” (Provider Block)**
```sh
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "mock"
  secret_key                  = "mock"
  s3_force_path_style         = true
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  endpoints {
    apigateway = "http://localhost:4566"
    lambda     = "http://localhost:4566"
    iam        = "http://localhost:4566"
    s3         = "http://localhost:4566"
  }
}
```
This tells Terraform, "Hey, use fake AWS (LocalStack) instead of real AWS."
It's like saying: “I want to play with fake Legos at home, not go to the Lego store.”

**Step 2: Create a Role for Lambda to Use (like a permission slip)**
```sh
resource "aws_iam_role" "lambda_exec_role" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}
```
This is like giving your Lambda function a backstage pass so it can run.
"IAM role" = permission slip for AWS stuff.

**Step 3: Create Your Lambda Function**
```sh
resource "aws_lambda_function" "hello_lambda" {
  function_name = "hello-lambda"
  filename      = "lambda.zip"
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  role          = aws_iam_role.lambda_exec_role.arn
  source_code_hash = filebase64sha256("lambda.zip")
}
```
"Here’s my function named hello-lambda, inside lambda.zip."
handler = "index.handler" means: "Look in the index.js file for a function called handler."
It's using Node.js version 18.
And it's using that permission slip from earlier.

**Step 4: Make an API (a public gate to access Lambda)**
```sh
resource "aws_api_gateway_rest_api" "example_api" {
  name = "example-api"
}
```
This creates a REST API called example-api.
It’s like setting up a front door to your backend.

**Step 5: Make an API Path: /hello**
```sh
resource "aws_api_gateway_resource" "proxy" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  parent_id   = aws_api_gateway_rest_api.example_api.root_resource_id
  path_part   = "hello"
}
```
Adds a door called /hello to your API.
Like saying: "When people visit /hello, do something."

**Step 6: Allow GET requests at /hello**
```sh
resource "aws_api_gateway_method" "get_proxy" {
  rest_api_id   = aws_api_gateway_rest_api.example_api.id
  resource_id   = aws_api_gateway_resource.proxy.id
  http_method   = "GET"
  authorization = "NONE"
}
```
This says: “Let anyone do a GET /hello request — no password needed.”

**Step 7: Connect the Door to Lambda**
```sh
resource "aws_api_gateway_integration" "lambda" {
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  resource_id = aws_api_gateway_resource.proxy.id
  http_method = aws_api_gateway_method.get_proxy.http_method

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.hello_lambda.invoke_arn
}
```
When someone hits the /hello endpoint, send the request to the Lambda function.
This is like putting a pipe from your API door straight to your Lambda function.

**Step 8: Allow API Gateway to Run Lambda**
```sh
resource "aws_lambda_permission" "api_gw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello_lambda.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.example_api.execution_arn}/*/*"
}
```
This is like saying: “Yes, API Gateway is allowed to run this Lambda function.”

**Step 9: Deploy Your API**
```sh
resource "aws_api_gateway_deployment" "example" {
  depends_on = [aws_api_gateway_integration.lambda]
  rest_api_id = aws_api_gateway_rest_api.example_api.id
  stage_name  = "dev"
}
```

This actually makes the API "live" at the /dev stage.
Think of it like pushing a "publish" button.

---
