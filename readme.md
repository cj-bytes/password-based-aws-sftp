# Password Based SFTP using AWS
### 1. Create bucket an S3 bucket
In the example below, I used `alpha-sftp-test-bucket`

### 2. Create IAM Role that will allow AWS Transfer to connect with an S3 Bucket
Establish Trust Relationship
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "transfer.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
Attach the following custom policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListingOfUserFolder",
            "Action": [
                "s3:ListBucket", 
                "s3:GetBucketLocation"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::alpha-sftp-test-bucket"
            ]
        },
        {
            "Sid": "HomeDirObjectAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:DeleteObjectVersion",
                "s3:GetObjectVersion",
                "s3:GetObjectACL",
                "s3:PutObjectACL"
            ],
            "Resource": "arn:aws:s3:::alpha-sftp-test-bucket/*"
        }
    ]
}
```

### 3. Create a lambda function.
Create a node.js lambda function with the following code. This resource will be referenced later

```
exports.handler = (event, context, callback) => {
    const expectedUserName = "sample-user",
        expectedPassword = "s@mpl3passw0rd",
        expectedIPs = ["192.168.1.1", "192.168.1.2"];

    const authenticatedResponse = {
        Role: "arn:aws:iam::123456789012:role/transfer-family-sftp-role",  //iam role with access to s3 and has trust established
        HomeDirectory: "/alpha-sftp-test-bucket" //bucket name
    };


    let isUsernameMatched = expectedUserName === event.username;
    let isPasswordMatched = expectedPassword === event.password;
    let isWhitelistedIp = expectedIPs.some(ip => ip == event.sourceIp);
    let hasValidCredentials = isUsernameMatched && isPasswordMatched && isWhitelistedIp;

    let response = hasValidCredentials ? authenticatedResponse : {};
    callback(null, response);
};
```

### 4. Create the AWS Transfer Familly Server
In the create wizard, refer to the listing below on what to select:
Select the protocols you want to enable
- SFTP 

Identity Provider for SFTP, FTPS, or FTP
- Custom Identity, Use AWS Lambda to connect your identity provider. Then reference the lambda that we created in the prior step

Endpoint configuration
- Public

Domain
- S3

Then leave the rest as default.

### 5. Update lambda created earlier to allow invocation from AWS Transfer Family via resource policy
Go to `Configuration` -> `Permissions` -> `Resource-based policy`
Then select the following:
- Policy statement
`AWS Service`
- Service
`Other`
- Statement ID
`AllowTransferInvocation`  (can be anything actually)
- Principal
`transfer.amazonaws.com`
- Source ARN  
The arn format is as follows 
`arn:aws:transfer:${Region}:${Account}:server/${ServerId}` See sample value below:
 ```
 arn:aws:transfer:ap-southeast-1:123456789091:server/s-a1234aa5ae6789010
 ```
- Action 
`lambda:InvokeFunction`


### 6. Test your Transfer server by using the hardcoded credentials on the function

You should get something similar to the image below
![image](https://user-images.githubusercontent.com/73715060/186577822-25b60725-1374-4696-b5c0-2e87a2a4431b.png)




