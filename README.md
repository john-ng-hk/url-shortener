# URLShortener

## Table of Contents
  - [Description](#Description)
  - [Architecture Diagram](#architecture-diagram)
  - [Detailed Steps](#detailed-steps)
    - [Implementation](#implementation)
    - [Deplyoment](#deplyoment)
    - [Usage](#usage)
  - [Cost](#cost)
  - [Limitations](#limitations)
  - [Potential Improvements](#potential-improvements)

## Description
The URL Shortener provides all the features and satisfied all requirements from Functional requirements, Infrastructure requirements and Optional Enhancements (Bonus). URLs are auto-generated, and expire after 10 minutes upon access. Documentation for Optional Enhancements (Bonus) are also included in this README.

URL Shortener Web Interface: https://d1y7ljm6v4i0aq.cloudfront.net/admin

## Architecture Diagram
The URL shortener adopted serverless architecture to create a scalable and efficient application, allowing the client to manage as little infrastructure as possible, while leveraging managed AWS services like AWS Lambda, AWS API Gateway, S3, DynamoDB, and CloudFront. The architecture also features a CI/CD pipeline from GitHub to AWS by utilizing GitHub actions.
![Diagram](/Diagram/URL-Shortener-AWS-Architecture-Diagram.drawio.png)

## Detailed Steps
### Step 1. Coding the infrastructures
As most services involved in the architecture built with Infrastructure as Code (Iac), this section will go through the infrastrucutre created with the CloudFormation template `stack.yaml`.

Below is the list of resources this template creates:
#### S3BucketForURLs 
Type: AWS::S3::Bucket  
Usage: stores all the redirection objects
#### S3InvokeLambdaPermission 
Type: AWS::Lambda::Permission  
Usage: Lambda resource-based policy to allow invocation from S3BucketForURLs
#### DynamoDBForObj 
Type: AWS::DynamoDB::Table  
Usage: stores last accessed time of the redirection objects
#### LambdaShortener 
Type: AWS::Lambda::Function  
Usage: Shorten the client's url with randomized suffix or custom suffix if given, return the complete shortened URL
#### LambdaRedirector 
Type: AWS::Lambda::Function  
Usage: Redirect the client's url short URL by finding the redirection object in S3BucketForURLs
#### LambdaExecRole 
Type: AWS::IAM::Role  
Usage: IAM Role for LambdaShortener and LambdaRedirector
#### LambdaObjDeleteExecRole 
Type: AWS::IAM::Role  
Usage: IAM Role for LambdaShortener and LambdaObjDelete
#### LambdaObjCountdown 
Type: AWS::Lambda::Function  
Usage: Store the short URL link last accessed time to DynamoDBForObj, and schedule when to delete it by sending a eventbridge cron rule to invoke LambdaObjDelete.
#### LambdaObjDelete 
Type: AWS::Lambda::Function  
Usage: Delete the target URL object from DynamoDBForObj when invoked
#### CloudFrontDistrib 
Type: AWS::CloudFront::Distribution  
Usage: A Content Delivery Network (CDN) service for simple friendly domain name, and wraps all the other services behind it.
#### LambdaShortenerInvokePermission 
Type: AWS::Lambda::Permission  
Usage: Lambda resource-based policy to allow invocation from URLShortenerAPI
#### LambdaRedirectorInvokePermission 
Type: AWS::Lambda::Permission  
Usage: Lambda resource-based policy to allow invocation from URLShortenerAPI
#### LambdaObjDeleteInvokePermission 
Type: AWS::Lambda::Permission  
Usage: Lambda resource-based policy to allow invocation from eventbridge events
#### URLShortenerAPI 
Type: AWS::ApiGateway::RestApi  
Usage: The API Gateway to serve response from the Lambda's backend for short url generation and redirection. Also serves the admin page and custom error page from it mock response.
#### URLShortenerAPIDeployment 
Type: AWS::ApiGateway::Deployment  
Usage: The deployment for URLShortenerAPI
#### URLShortenerAPIStage 
Type: AWS::ApiGateway::Stage  
Usage: The stage for URLShortenerAPI

#### Outputs
The list of outputs this template exposes:

#### BucketName 
Description: Amazon S3 bucket name holding short URLs redirect objects. Note: the bucket will not be deleted when you delete this template.  

#### ConnectURL 
Description: URL to connect to the admin page of the URL Shortener.

### Step 2. Settign the CI/CD pipeline
After having all the above services coded in `stack.yaml`, we proceed to setup a CI/CD pipeline by utilizing GitHub Actions workflow to allow automated creation of infrastructures in AWS whenever we commit changes to GitHub.

GitHub Action workflows must access resources in our AWS account. So we create a pair of AWS secret access key, with the permission needed to create all the above infrastructures.

Afer setting the proper permissions, we proceed to create `deploy.yaml` with configured AWS credentials, which will trigger `stack.yaml` deployment to AWS whenever a there is a push on the GitHub repository `main` branch.

When the workflow is triggered, it will deploy to AWS by making a change set. The workflow will then continue to run until the stack is deployed successfuly or failed.

### Step 3. Deplyoment
By triggering the GitHub workflow (either manually or automatically by pushing the `main`), URL Shortener will then be automactically deployed in the ap-east-1 (Hong Kong) region of AWS after a couple of minutes. After it is deployed, we can go to the URL Shortener portal by clicking the `ConnectUrl` in the CloudFormation output on AWS.



#### IP blocking and Additional Security Masures
WAF is used for IP blocking and for additiona security measures. For WAF, it is not included in `stack.yaml` as the rules may forbidden users from accessing URL Shortener without a IP from the range 218.189.44.128 - 218.189.44.255, preventing users from accessing URL Shortener. Hence, manual deployment is needed for WAF. 

To deploy WAF, first we create a IP set to only allow users with IP range from 218.189.44.128 - 218.189.44.255 to submit/access short URLs.
![Create IP Set](/WAF%20deployment/create-ip-set.png)

Next, we create a web ACL and associate it our Cloudfront distribution
![Associate CF](/WAF%20deployment/Associate%20CF.png)
Then, we create a WAF rule and bind the IP set to it.
![Create web acl](/WAF%20deployment/create-waf-web-acl.jpeg)

Finally, we can see that our CloudFront distribution is protected by WAF and cannot be accessed without IP from the range 218.189.44.128 - 218.189.44.255
![CloudFront Protected](/WAF%20deployment/cf-protected.png)
![No access](/WAF%20deployment/no-access.png)

##### Additional Security Masures
For additional security measures, AWS managed rules have been deployed to enhance the overall security:

1. `AWS-AWSManagedRulesAmazonIpReputationList`  
This group contains rules that are based on Amazon threat intelligence. This is useful if you would like to block sources associated with bots or other threats.

2. `AWS-AWSManagedRulesAnonymousIpList`  
This group contains rules that allow you to block requests from services that allow obfuscation of viewer identity. This can include request originating from VPN, proxies, Tor nodes, and hosting providers. This is useful if you want to filter out viewers that may be trying to hide their identity from your application.

![additional security measures](/WAF%20deployment/Additional%20Security%20Measure.png)

### Step 4. Usage
To use the portal, simply go the admin page and type the orginal URL in the provided field. User may also provide their own suffix of 5 characters (a-z, 0-9). If suffix provided, one will be generated randomly and the short URL will be returned. Then the user may use the generated short URL and URL Shortener will redirect the user to the original long URL. 

Should the user provided a suffix that already in use, the portal will prompt the user to provide another suffix.

The generated short URL link will be expired if it has not been accessed for 10 min.
## Cost
Assumptions:
1000 short urls created per month, each viewed by 1000 users. So 1000x1000 = 1,000,000 requests/month.

Monthy cost: 25.31 USD  
Yearly cost: 301.56 USD  
No upfront cost.

Detailed estimation link: https://calculator.aws/#/estimate?id=8b8acbf2bd7507af72718e6150efa700fa2e8c9b

## Limitations
### Limitations on concurrent users
The limitation is 1,000 concurrent users with the current proposed setup. The bottleneck is Lambda, which is supporting only 1,000 concurrent concurrent executions with the defualt quota.

### Scaling strategies
The URL Shortener scales well with the proposed fully serverless architecutre. However, to support more than 1,000 concurrent users, the default quota of Lambda it must be increased by [Requesting a quota increase](https://docs.aws.amazon.com/servicequotas/latest/userguide/request-quota-increase.html).

## Potential Improvements
1. Redirect uesr back to homepage if a the link is invalid.

2. Setup custom DNS domain name to create a really short URL instead of using cloudfront domain.

3. Setup mechanisms like Captcha to deter bot attacks and spam on the service.

4. Setup access control such as Cognito to the portal for secure authentication.

5. Implement loggings such as CloudWatch to track usage and errors.

6. Setup unit tests in the CI/CD pipeline.
