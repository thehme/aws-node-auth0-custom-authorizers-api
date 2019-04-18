# API Gateway Custom Authorizer Function + Auth0

This is an extension of the serveless example: https://github.com/serverless/examples/tree/master/aws-node-auth0-custom-authorizers-api

This is an example of how to protect API endpoints with [auth0](https://auth0.com/), JSON Web Tokens (jwt) and a [custom authorizer lambda function](https://serverless.com/framework/docs/providers/aws/events/apigateway#http-endpoints-with-custom-authorizers).

Custom Authorizers allow you to run an AWS Lambda Function before your targeted AWS Lambda Function. This is useful for Microservice Architectures or when you simply want to do some Authorization before running your business logic.

### [View live demo](https://aws-custom-authorizer-auth0-test.s3-us-west-2.amazonaws.com/index.html)

## Use cases

- Protect API routes for authorized users
- Rate limiting APIs

## Setup

1. `npm install` json web token dependencies

2. `npm install --save-dev serverless-single-page-app-plugin` to install the plugin needed to upload static files to S3

3. Copy `node_modules/serverless-single-page-app-plugin` into the root folder

4. Register it in your `serverless.yml` file, as a plugin:

```
plugins:
  - serverless-single-page-app-plugin
```

5. Set an `s3LocalPath` custom variable to the folder where the files that will be synched to S3 are located, e.g. 'dist/':

```
custom:
  s3LocalPath: dist/
```

6. Add appropriately-named resources (Bucket, BucketPolicy and Distribution) and Outputs:

```
resources:
  Resources:
    ## Specifying the S3 Bucket
    WebAppS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        WebsiteConfiguration:
          IndexDocument: index.html
          ErrorDocument: index.html
    ## Specifying the policies to make sure all files inside the Bucket are avaialble to CloudFront
    WebAppS3BucketPolicy:
      Type: AWS::S3::BucketPolicy
      Properties:
        Bucket:
          Ref: WebAppS3Bucket
        PolicyDocument:
          Statement:
            - Sid: PublicReadGetObject
              Effect: Allow
              Principal: "*"
              Action:
              - s3:GetObject
              Resource: 
                Fn::Join: [
                  "", [
                    "arn:aws:s3:::",
                    { "Ref": "WebAppS3Bucket" },
                    "/*"
                  ]
                ]
    ## Specifying the CloudFront Distribution to server your Web Application
    WebAppCloudFrontDistribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - DomainName:
                Fn::Join: [
                  "", [
                    { "Ref": "WebAppS3Bucket" },
                    ".s3.amazonaws.com"
                  ]
                ]
              ## An identifier for the origin which must be unique within the distribution
              Id: WebApp
              CustomOriginConfig:
                HTTPPort: 80
                HTTPSPort: 443
                OriginProtocolPolicy: https-only
              ## In case you want to restrict the bucket access use S3OriginConfig and remove CustomOriginConfig
              # S3OriginConfig:
              #   OriginAccessIdentity: origin-access-identity/cloudfront/E127EXAMPLE51Z
          Enabled: 'true'
          ## Uncomment the following section in case you are using a custom domain
          # Aliases:
          # - mysite.example.com
          DefaultRootObject: index.html
          ## Since the Single Page App is taking care of the routing we need to make sure ever path is served with index.html
          ## The only exception are files that actually exist e.h. app.js, reset.css
          CustomErrorResponses:
            - ErrorCode: 404
              ResponseCode: 200
              ResponsePagePath: /index.html
          DefaultCacheBehavior:
            AllowedMethods:
              - DELETE
              - GET
              - HEAD
              - OPTIONS
              - PATCH
              - POST
              - PUT
            ## The origin id defined above
            TargetOriginId: WebApp
            ## Defining if and how the QueryString and Cookies are forwarded to the origin which in this case is S3
            ForwardedValues:
              QueryString: 'false'
              Cookies:
                Forward: none
            ## The protocol that users can use to access the files in the origin. To allow HTTP use `allow-all`
            ViewerProtocolPolicy: redirect-to-https
          ## The certificate to use when viewers use HTTPS to request objects.
          ViewerCertificate:
            CloudFrontDefaultCertificate: 'true'
          ## Uncomment the following section in case you want to enable logging for CloudFront requests
          # Logging:
          #   IncludeCookies: 'false'
          #   Bucket: mylogs.s3.amazonaws.com
          #   Prefix: myprefix

  ## In order to print out the hosted domain via `serverless info` we need to define the DomainName output for CloudFormation
  Outputs:
    WebAppS3BucketOutput:
      Value:
        'Ref': WebAppS3Bucket
    WebAppCloudFrontDistributionOutput:
      Value:
        'Fn::GetAtt': [ WebAppCloudFrontDistribution, DomainName ]
```

7. Setup an [auth0 application](https://auth0.com/docs/applications).

8. Get your `Client ID` (under `applications->${YOUR_APP_NAME}->settings`) and plugin your `AUTH0_CLIENT_ID` in a new file called `secrets.json` (based on `secrets.example.json`).

9. Get your `public key` (under `applications->${YOUR_APP_NAME}->settings->Show Advanced Settings->Certificates->DOWNLOAD CERTIFICATE`). Download it as `PEM` format and save it as a new file called `public_key`, no extension

10. Deploy the service with `serverless deploy` or `sls deploy` and grab the public and private endpoints.

11. Plugin your `AUTH0_CLIENT_ID`, `AUTH0_DOMAIN`, and the `PUBLIC_ENDPOINT` + `PRIVATE_ENDPOINT` from aws in top of the `frontend/app.js` file.

  ```js
  /* frontend/app.js */
  // replace these values in app.js
  const AUTH0_CLIENT_ID = 'your-auth0-client-id-here';
  const AUTH0_DOMAIN = 'your-auth0-domain-here.auth0.com';
  const PUBLIC_ENDPOINT = 'https://your-aws-endpoint-here.amazonaws.com/dev/api/public';
  const PRIVATE_ENDPOINT = 'https://your-aws-endpoint-here.us-east-1.amazonaws.com/dev/api/private';
  ```

12. Configure the `Allowed Callback URL` and `Allowed Origins` in your auth0 client in the [auth0 dashboard](https://manage.auth0.com).

For this part, I was not sure what to use, so I just used https://google.com for `Allowed Callback URL` and http://localhost:3000 for `Allowed Origins`. I did this in case it caused something to break if I did not have anything else in place.

## Sync to S3

At this point, the static files need to be pushed to S3 using `serverless syncToS3`, but for some reason this is not working for me. Since the plug is running the `aws` command to do this, I decided to test it directly:

`aws s3 sync app/ s3://yourBucketName123/`

Once I did this, it outputted something like this:

```
> aws s3 sync app s3://aws-custom-authorizer-auth0-test
upload: app/app.js to s3://aws-custom-authorizer-auth0-test/app.js
upload: app/app.css to s3://aws-custom-authorizer-auth0-test/app.css
upload: app/index.html to s3://aws-custom-authorizer-auth0-test/index.html
```

You can confirmed the files were uploaded by checking your bucket in S3 at https://s3.console.aws.amazon.com/s3/buckets. Also, to view the deployed app, check the CloudFront distrution at https://console.aws.amazon.com/cloudfront and find the domain name. Put that in your URL and you should see the app loaded on this URL.

## Custom authorizer functions

[Custom authorizers functions](https://aws.amazon.com/blogs/compute/introducing-custom-authorizers-in-amazon-api-gateway/) are executed before a Lambda function is executed and return an Error or a Policy document.

The Custom authorizer function is passed an `event` object as below:

```javascript
{
  "type": "TOKEN",
  "authorizationToken": "<Incoming bearer token>",
  "methodArn": "arn:aws:execute-api:<Region id>:<Account id>:<API id>/<Stage>/<Method>/<Resource path>"
}
```