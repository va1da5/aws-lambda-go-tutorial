# Deploying a Golang REST API to AWS Lambda

Here's a tutorial on deploying a Golang Gin-based REST API to a Lambda function using the [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter/tree/main), all without changing your application code. This approach can also be applied to other types of REST APIs or web applications.

The application code can remain unchanged thanks to the [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter/tree/main), which acts as an [additional layer](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) between the user and your web application. This [layer](https://community.aws/content/2fJxs6oeXINRtG18bXmou7nea5i/adding-flexibility-to-your-deployments-with-lambda-web-adapter#how-lambda-web-adapter-works-) translates Lambda events into valid HTTP requests that your application recognizes. However, please be aware that this may introduce some unwanted latency in application responses, so it's important to evaluate whether this setup meets your requirements.

## Getting Started

These instructions will help you create a new Go Gin project and guide you on how to deploy it to AWS Lambda.

1. Create a new project using [go-blueprint](https://github.com/Melkeydev/go-blueprint) utility. You can find all of the utility options using [dedicated web UI](https://go-blueprint.dev/):

   ```bash
   go-blueprint create --name aws-lambda-go-tutorial --framework gin --driver none --git skip
   ```

2. Once the necessary changes are made to the API source code, it can be bundled for AWS Lambda. It's [recommended](https://aws.amazon.com/blogs/apn/comparing-aws-lambda-arm-vs-x86-performance-cost-and-analysis-2/) to compile the application for ARM64 instruction:

   ```bash
   GOOS=linux GOARCH=arm64 go build -o main cmd/api/main.go
   zip main.zip main
   ```

3. Configure Lambda execution role

   ```bash
   # creates temporary trust policy document
   cat > trust-policy.json << EOF
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Principal": {
                   "Service": "lambda.amazonaws.com"
               },
               "Action": "sts:AssumeRole"
           }
       ]
   }
   EOF

   # creates role. Please note the role ARN for later use.
   aws iam create-role \
       --role-name LambdaExecutionRole \
       --assume-role-policy-document file://trust-policy.json \
       --max-session-duration 7200

   # removes the trust policy file
   rm trust-policy.json

   # finds the ARN for required IAM policy
   aws iam list-policies --query 'Policies[?PolicyName==`AWSLambdaBasicExecutionRole`]'

   # attaches policy to the newly created role for lambda function
   aws iam attach-role-policy \
       --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole \
       --role-name LambdaExecutionRole
   ```

4. Once the bundle is ready, you can create the function using the AWS CLI. Please note that the layer ARN is region-specific, so be sure to update the value based on the region where the function is being deployed.

   ```bash
   aws lambda create-function --function-name go-api-tutorial \
   --runtime provided.al2023 --handler main \
   --architectures arm64 \
   --role arn:aws:iam::381492052417:role/LambdaExecutionRole \
   --layers "arn:aws:lambda:us-east-1:753240598075:layer:LambdaAdapterLayerArm64:24" \
   --environment "Variables={PORT=8000,AWS_LAMBDA_EXEC_WRAPPER=/opt/bootstrap,GIN_MODE=release}" \
   --zip-file fileb://main.zip

   # optional: update existing function with the latest version of the app
   aws lambda update-function-code --function-name go-api-tutorial \
   --zip-file fileb://main.zip
   ```

5. [Configure Lambda function URL](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-function-url-config.html). Successfully executing the command below will configure the URL for your Lambda function and eliminate the need for AWS API Gateway. Please note that the configuration is not secure by default, so be sure to adjust the settings as needed.

   ```bash
   aws lambda create-function-url-config --function-name go-api-tutorial \
   --auth-type NONE \
   --invoke-mode BUFFERED \
   --cors "AllowOrigins=*,AllowMethods=*,MaxAge=60"
   ```

6. If the created resources are no longer needed, you can easily remove them using the following commands:

   ```bash
   # deletes URL configuration for the Lambda function
   aws lambda delete-function-url-config --function-name go-api-tutorial

   # deletes the Lambda function
   aws lambda delete-function --function-name go-api-tutorial

   # deletes the IAM role
   aws iam delete-role --role-name LambdaExecutionRole
   ```

## References

- [go-blueprint](https://github.com/Melkeydev/go-blueprint)
- [Comparing AWS Lambda Arm vs. x86 Performance, Cost, and Analysis](https://aws.amazon.com/blogs/apn/comparing-aws-lambda-arm-vs-x86-performance-cost-and-analysis-2/)
- [Selecting and configuring an instruction set architecture for your Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/foundation-arch.html)
- [Run any web app on Lambda | Serverless Office Hours](https://www.youtube.com/watch?v=ArsTZ2y7u80)
