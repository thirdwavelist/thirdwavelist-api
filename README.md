# api.thirdwavelist

## Setup

This setup uses the following Amazon Web Services:
- S3
- CloudFormation
- ApiGateway
- DynamoDB

The API is based on: Swagger / OpenAPI Specification



1. Clone this repository

```bash
git clone git@github.com:hipinc/api.thirdwavelist.git
cd api.thirdwavelist/
```

2. Create the lambda code file (`lambda.zip`)

```bash
npm install --production
./bundle.sh
```

3. Create an S3 bucket called `api.thirdwavelist` in the US East (N. Virginia, `us-east-1`) region and upload the `lambda.zip` file

**Note: At this point please ensure you have set up `awscli` and ran `aws configure` before continuing.**

```bash
aws s3 mb s3://api.thirdwavelist
aws s3 cp lambda.zip s3://api.thirdwavelist/lambda.zip
```

4. Create cloudformation stack

```bash
aws cloudformation create-stack --stack-name thirdwavelist-apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,ParameterValue=api.thirdwavelist
```

5. Wait until the stack is created (`CREATE_COMPLETE`)

```bash
aws cloudformation wait stack-create-complete --stack-name thirdwavelist-apigateway
```

6. Replace **all nine occurrences** of `$AWSRegion` in `swagger.json` with the region that you are creating your API and Lamdba in

```bash
cp swagger.json swagger.json.bak
sed 's/$AWSRegion/us-east-1/g' swagger.json.bak > swagger.json

```

7. Get the `LambdaArn`

```bash
aws cloudformation describe-stacks --stack-name thirdwavelist-apigateway --query "Stacks[0].Outputs"
```

8. Replace **all four occurrences** of `$LambdaArn` in `swagger.json` with the ARN from the stack output above (e.g. look for the XXX and replace it with the output of the previous command)

```bash
sed 's/$LambdaArn/XXX/g' swagger.json > swagger.json.temp; mv -f swagger.json.temp swagger.json
```

9. Deploy the API Gateway

**Note: make sure you have an up-to-date version (`aws --version`) of the AWS CLI >= 1.10.18. Learn more [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html).**

```bash
aws apigateway import-rest-api --fail-on-warnings --body file://swagger.json
```

10. Update the CloudFormation template to set the `ApiId` parameter (replace `$ApiId` with the `id` output from above)

```bash
aws cloudformation update-stack --stack-name thirdwavelist-apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,UsePreviousValue=true ParameterKey=S3Key,UsePreviousValue=true ParameterKey=ApiId,ParameterValue=$ApiId
```

11. Deploy to Stage v1 (replace `$ApiId`)

```bash
aws apigateway create-deployment --rest-api-id $ApiId --stage-name v1
```

12. (Optional) Set the `$ApiGatewayEndpoint` environment variable (replace `$ApiId`)

```bash
export ApiGatewayEndpoint="$ApiId.execute-api.us-east-1.amazonaws.com/v1"
```

## Use the RESTful API

The following examples assume that you replace `$ApiGatewayEndpoint` with `$ApiId.execute-api.us-east-1.amazonaws.com`

1. Create a cafe and POST it

```bash
curl -vvv -d '{"name": "The best coffeeshop", "thumbnail": "http://lorempixel.com/200/200/city/1", "address": "Vaci street 1"}' -H "Content-Type: application/json" https://$ApiGatewayEndpoint/cafe
```

2. List cafes with a GET

```bash
curl -vvv https://$ApiGatewayEndpoint/cafe
```

## Teardown

1. Delete API Gateway (replace `$ApiId`)

```bash
aws apigateway delete-rest-api --rest-api-id $ApiId
```

2. Delete CloudFormation stack

```bash
aws cloudformation delete-stack --stack-name thirdwavelist-apigateway
```

3. Delete S3 bucket (replace `$S3Bucket`)

```bash
aws s3 rb --force s3://api.thirdwavelist
```