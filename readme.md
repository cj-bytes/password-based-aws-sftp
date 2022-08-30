# Password Based SFTP using AWS
## 1. Create an S3 bucket and Parameter Store Variables
In the example below, I used `alpha-sftp-test-bucket`

Add to Parameter store the following values. You will reference them later. Use secure string where necessary.
```
'/sftp-username'  //sample-user

'/sftp-password'    //s@mpl3passw0rd

'/sftp-allowed-ips',   //192.168.1.1, 192.168.1.2    seperate it with a comma

'/sftp-role-arn',   //iam role with access to s3 and has trust established i.e. arn:aws:iam::123456789012:role/transfer-family-sftp-role  

'/sftp-bucket',     //bucket name like  /alpha-sftp-test-bucket
```

## 2. Create IAM Role that will allow AWS Transfer to connect with an S3 Bucket
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

## 3. Create a lambda function
Create a node.js lambda function with the following code. This resource will be referenced later

```
exports.handler = async (event, context, callback) => {


    let parameters = await getParameters([
         '/sftp-username',  //sample-user
         '/sftp-password',    //s@mpl3passw0rd
         '/sftp-allowed-ips',   //     192.168.1.1, 192.168.1.2    seperate it with a comma
         '/sftp-role-arn',   //iam role with access to s3 and has trust established i.e.   arn:aws:iam::123456789012:role/transfer-family-sftp-role  
         '/sftp-bucket',     //bucket name like  /alpha-sftp-test-bucket
     ]);
 
  
     const expectedUserName = parameters.find(x=> x.Name == "/sftp-username").Value,  
         expectedPassword = parameters.find(x=> x.Name == "/sftp-password").Value,  
         expectedIPs = parameters.find(x=> x.Name == "/sftp-allowed-ips").Value.split(",").map(x => x.trim()),  
         Role = parameters.find(x=> x.Name == "/sftp-role-arn").Value,     
         HomeDirectory = parameters.find(x=> x.Name == "/sftp-bucket").Value; 
  
 
     const authenticatedResponse = {
         Role,
         HomeDirectory
     };
 
     let isUsernameMatched = expectedUserName === event.username;
     let isPasswordMatched = expectedPassword === event.password;
     let isWhitelistedIp = expectedIPs.some(ip => ip == event.sourceIp);
     let hasValidCredentials = isUsernameMatched && isPasswordMatched && isWhitelistedIp;
 
     let response = hasValidCredentials ? authenticatedResponse : {};
     callback(null, response);
 };
 
 
 
 function getParameters(parameterNames) {
     const ssm = new (require('aws-sdk/clients/ssm'))();
 
     return new Promise(((resolve, reject) => {
         var params = {
             Names: parameterNames, 
             WithDecryption: true
         };
 
 
         ssm.getParameters(params, function (err, data) {
             if (err) {
                 console.log(err, err.stack);
                 reject();
             }
             else { 
                 resolve(data.Parameters)
             }
         });
     }))
 }
```
Allowing reading of Systems Manager Parameter Store by attaching an inline policy to the lambda execution IAM Role
![image](https://user-images.githubusercontent.com/73715060/187477441-569a1fdc-06a9-497f-b6cf-dd418b83e902.png)

You may refer to this inline policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "ssm:GetParameters", 
            "Resource": "arn:aws:ssm:ap-southeast-1:123456789012:parameter/*"
        }
    ]
}
```

Test the lambda you created by using the following test event
```
{
    "username": "sample-user",
    "password": "s@mpl3passw0rd",
    "protocol": "SFTP",
    "serverId": "s-abcd123456",
    "sourceIp": "192.168.1.1"
}
```

You should get the following response\
![image](https://user-images.githubusercontent.com/73715060/187478029-597c4028-1af6-4967-b764-db13cca61f6d.png)


## 4. Create the AWS Transfer Familly Server
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

## 5. Update lambda created earlier to allow invocation from AWS Transfer Family via resource policy
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


## 6. Test your Transfer server by using the hardcoded credentials on the function

You should get something similar to the image below
![image](https://user-images.githubusercontent.com/73715060/186577822-25b60725-1374-4696-b5c0-2e87a2a4431b.png)




