# cfn-secret-provider

## FORKED
This repo is forked from  [binxio/cfn-resource-provider](https://github.com/binxio/cfn-resource-provider) to support:
* Deploying the secret provider into your account entirely from code under your control (rather than deploying from the prebuilt binxio-public s3 bucket)
* Support for Secrets Manager secrets rotation
I will work to get these changes merged upstream.

-----------------------------------------------------------------------------------------------------------------------

A CloudFormation custom resource provider for managing secrets, private keys and EC2 key pairs.

One of the biggest problems I encounter in creating immutable infrastructures, is dealing with secrets. Secrets must always be different per
environment and therefore parameterized. As we automated all the things passwords often end up in parameter files and have to pass them around 
to people and applications: This is not a good thing. With this Custom CloudFormation Resource we put an end to that. Secrets are generated, 
stored in the EC2 parameter store and access to the secrets can be controlled through security policies.

## How do I generate a secret?
It is quite easy: you specify a CloudFormation resource of the [Custom::Secret](docs/Custom%3A%3ASecret.md), as follows:

```json
  "Resources": {
    "DBPassword": {
      "Type": "Custom::Secret",
      "Properties": {
        "Name": "/demo/PGPASSWORD",
        "KeyAlias": "alias/aws/ssm",
        "Alphabet": "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789",
        "Length": 30,
        "ReturnSecret": true,
        "ServiceToken": { "Fn::Join": [ ":", [ "arn:aws:lambda", { "Ref": "AWS::Region" }, { "Ref": "AWS::AccountId" }, "function:binxio-cfn-secret-provider" ] ]
        }
      }
    }
  }
```
After the deployment, a 30 character random string can be found in the EC Parameter Store with the name `/demo/PGPASSWORD`.

If you need to access the secret in your cloudformation module, you need to specify `ReturnSecret` and reference it as the attribute `Secret`.

```json
        "MasterUserPassword": { "Fn::GetAtt": [ "DBPassword", "Secret" ]}
```

## How do I add a private key?
In the same manner you can specify a RSA private key as a CloudFormation resource of the [Custom::RSAKey](docs/Custom%3A%3ARSAKey.md):

```json
  "Resources": {
    "PrivateKey": {
      "Type": "Custom::RSAKey",
      "Properties": {
        "Name": "/demo/private-key",
        "KeyAlias": "alias/aws/ssm",
        "ServiceToken": { "Fn::Join": [ ":", [ "arn:aws:lambda", { "Ref": "AWS::Region" }, { "Ref": "AWS::AccountId" }, "function:binxio-cfn-secret-provider" ] ]
        }
      }
    }
  }
```
After the deployment, a the newly generated private key can be found in the EC2 Parameter Store with the name `/demo/private-key`.

If you need to access the public key of the newly generated private key, you can reference it as the attribute `PublicKey`.  Most likely, 
you would use this in the [Custom::KeyPair](docs/Custom%3A%3AKeyPair.md) resource, to create a EC2 key pair:

```json
    "KeyPair": {
      "Type": "Custom::KeyPair",
      "DependsOn": "CustomPrivateKey",
      "Properties": {
        "Name": "CustomKeyPair",
        "PublicKeyMaterial": { "Fn::GetAtt": [ "PrivateKey", "PublicKey" ] },
        "ServiceToken": { "Fn::Join": [ ":", [ "arn:aws:lambda", { "Ref": "AWS::Region" }, { "Ref": "AWS::AccountId" }, "function:binxio-cfn-secret-provider" ] ] }
      }
```
This will create the ec2 key pair for you named `CustomKeyPair`, based on the generated private key. Now private key is securely stored in the EC2 Parameter Store and the public key can be used to gain access to specific EC2 instances. See [Amazon EC2 Key Pairs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) for more information.


## Installation
To install these custom resources, type:

```sh
aws cloudformation create-stack \
	--capabilities CAPABILITY_IAM \
	--stack-name cfn-secret-provider \
	--template-body file://cloudformation/cfn-resource-provider.json 

aws cloudformation wait stack-create-complete  --stack-name cfn-secret-provider 
```

This CloudFormation template will use our pre-packaged provider from `s3://binxio-public-${AWS_REGION}/lambdas/cfn-secret-provider-0.12.0.zip`.


## Demo
To install the simple sample of the Custom Resource, type:

```sh
aws cloudformation create-stack --stack-name cfn-secret-provider-demo \
	--template-body file://cloudformation/demo-stack.json
aws cloudformation wait stack-create-complete  --stack-name cfn-secret-provider-demo
```

to validate the result, type:

```sh
aws ssm get-parameter --name /demo/PGPASSWORD --with-decryption
aws ssm get-parameter --name /demo/private-key  --with-decryption
aws ec2 --output text describe-key-pairs --key-names CustomKeyPair 
```

## Conclusion
With this solution: 

- secrets are generated per environment
- always stored encrypted in the parameter store 
- where access to the secrets is audited and controlled!

