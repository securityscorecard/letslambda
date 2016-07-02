# LetsLambda #

A python script that gets to renew your SSL certificates from AWS Lambda via DNS challenge using [Let's Encrypt](https://letsencrypt.org/) services. It stores your keys and certificates in a S3 bucket. If the keys don't exists, it generates them and re-uses them later (useful for [public key pinning](https://en.wikipedia.org/wiki/HTTP_Public_Key_Pinning)).

All in all, the script talks to [Let's Encrypt](https://letsencrypt.org/) and Amazon [Route53](https://aws.amazon.com/route53/) (for the DNS challenge), Amazon [S3](https://aws.amazon.com/s3/) and Amazon [IAM](https://aws.amazon.com/iam/) (to store your certificates) and Amazon [Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/). And optionally, Amazon [KMS](https://aws.amazon.com/kms/) can be used to encrypt your data in your S3 bucket.

## Configuration ##
The configuration file is based on YAML. It should be easy to understand by reviewing the provided configuration. Nonetheless, here is a short explanation of each configuration directive
```yaml
directory: https://acme-v01.api.letsencrypt.org/directory
base_path: letsencrypt/
delete_expired_cert: true
info:
  - mailto:myemail@example.com
domains:
  - name: www.example.com
    r53_zone: example.com
    countryName: FR
    reuse_key: true
    key_size: 2048
    base_path: letsencrypt/certificates/example.com/
    elbs:
      - name: elb_name
        region: ap-southeast-2
        port: 443
    cfs:
      - id: XXXXXXXXXXXXXX
      - id: YYYYYYYYYYYYYY
  - name: api.anotherexample.com
    r53_zone: anotherexample.com
    countryName: AU
    reuse_key: true
    base_path: letsencrypt/certificates/
    elbs:
      - name: elb_name_2
        region: ap-southeast-2
        port: 443
      - name: elb_name_3
        region: ap-southeast-1
        port: 443
  - name: old.example.com
    r53_zone: example.com
    countryName: FR
    reuse_key: false
    key_size: 4096
    elb: old_elb_name
    elb_port: 8443
    elb_region: us-east-1
```

### Configuring Let's encrypt ###
`directory`: The Let's Encrypt directory endpoint to use to request the certificate issuance. This is useful when you need to switch between staging and production. Possible values are:

 - `https://acme-v01.api.letsencrypt.org/directory` for production
 - `https://acme-staging.api.letsencrypt.org/directory` for development and tests

`base_path`: This defines the location in your S3 bucket where the Let's encrypt account key stored. It also serves at the default location to store per domain private keys and issued certificates. If not specified, the root (`/`) of your S3 bucket will be used instead.

`delete_expired_cert`: This defines whether or not expired server certificates stored in IAM should be removed. By default, an AWS account can store up to 20 server certificates making this resource quite limited. And since a server certificate can only be added or removed (not updated), the renewal process may easily pass the maximum allows limit. If unspecified the default is `false` (do not remove). Server certificates stored in the S3 bucket aren't affected regardless of this value. If the server certificate is linked to an AWS service (ELB or CloudFront), the deletion will fail.

`info`: The information to be used when the script is registering your account for the first time. You should provide a valid email or the registration may fail.

    info:
        - mailto:myemail@example.com

### Configuring your domains ###
Letslambda allows you to declare multiple host names that you wish to get a certificate for.

Each is declared under the `domains` list.

Here is the details for each domain.

`domains`: a list of domain information.

 - `- name`: The host name for which you want your certificate to be issued for.
 - `r53_zone`: the Route53 hosted zone name which contains the DNS entry for `name`.
 - `countryName`: This parameter is used for the `countryName` in the [Certificate Signing Request](https://en.wikipedia.org/wiki/Certificate_signing_request) (CSR). It's a 2 letters representation of the country name. It follows the [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) standard.
 - `kmsKeyArn`: Your KMS key arn to encrypt the Let's Encrypt account key and your certificate private keys. You may also use `AES256` for AWS managed at rest encryption. Default is `AES256`.
 - `reuse_key`: The Lambda function will try to reuse the same private key to generate the new CSR. This is useful if you ever want to use Public Key Pinning (Mobile App development) and yet want to renew your certificates every X months
 - `key_size`: Determine the private key size (in bits). Common values are `2048` or `4096`. Note that Amazon CloudFront [doesn't support certificates for keys longer than 2048](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/SecureConnections.html#CNAMEsAndHTTPS). If omitted, the default value is `2048` since it's secure and it maximises compatibility with the AWS services.
 - `base_path`: This defines the location in your S3 bucket where the domain private keys and issued certificate is saved. If not specified, it defaults to the global `base_path` (see above).

### Configuring your ELBs ###
You have 2 ways to get your server certificates deployed into one or more Elastic Load Balancer (ELB). However, this section is optional is you don't have any ELB.

The initial release of Letslambda used to support only one ELB. This is done as follows:
 - `elb`: Name of your Elastic Load Balancer.
 - `elb_region`: the region is which your ELB has been deployed in. Default is the Lambda local region.
 - `elb_port`: ELB listening port. If left unspecified, the default is `443` (HTTPS).

Only **one** ELB is supported when configured this way.

The newer and preferred way to declare the deployment of your server certificate into your ELB is done as follows:
 - `elbs` which is the start of your ELB list.

 And below, per ELB specific settings:
 - `- name` is for the name of your ELB. This parameter should come first in the ELB list.
 - `region` the AWS region code in which your ELB is deployed into. If missing, this defaults to the AWS region in which Letslambda runs.
 - `port` which represents the ELB listener port. This port must already be configure ahead. If omitted, the default value is `443` (HTTPS).

### Configuring your CloudFront distributions ###
Just like Elastic Load Balancers, LetsLambda supports one or more CloudFront distributions as part of the configuration file.
 - `cfs` which is the start of your CloudFront list. You may ommit this parameter if you don't have any CloudFront distribution.

And for each CloudFront distribution:
 - `- id` which represents your CloudFront distribution ID

Unlike other AWS services, CloudFront requires some time to be fully deployed. Usually about 30 minutes but this may vary. LetsLambda will __not__ updated your distribution configuration if it's not in a deployed state (where pending configuration changes are being deployed).

## Installation ##

This project relies on third party projects that requires some files to be compiled. Since AWS Lambda runs on Amazon Linux 64 bit ([ref](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)), it's important that you have such instance running to prepare your Lambda function and not a custom debian/ubuntu server as you may find some libraries incompatibilities).

    $> yum install libcffi-devel libffi-devel libyaml-devel gcc openssl-devel git
    $> virtualenv .env
    $> source .env/bin/activate
    $> pip install -r requirements.txt
    $> ./make_package.sh

Once this is done, all you have to do is to upload your lambda function to a S3 bucket.
    $> aws s3 cp letslambda.zip s3://bucket/

Alternatively, you may use the Amazon Management Console to upload your package from the comfort of your web browser.

And finally, let Amazon CloudFormation do the heavy job of deploying the LetsLambda function.

    $> aws cloudformation create-stack --stack-name letslambda --template-body  file://letslambda.json \
           --parameters ParameterKey=FnBucket,ParameterValue=bucket_name \
           ParameterKey=FnPath,ParameterValue=some/path/to/letslambda.zip \
           ParameterKey=Bucket,ParameterValue=bucket_name \
           ParameterKey=ConfigFile,ParameterValue=some/path/to/letslambda.yml \
           ParameterKey=Region,ParameterValue=eu-west-1 \
           ParameterKey=KmsEncryptionKeyArn,ParameterValue=arn:aws:kms:eu-central-1:123456789012:key/30df8784-b708-4bea-8506-b12cc04335a4 \
           --capabilities CAPABILITY_IAM

As a possible alternative, you may use the CloudFormation Management Console to deploy your Lambda function. Though, you should ensure that you deploy the IAM resources included in the template.

The above parameters are:
 - `FnBucket`: S3 Bucket name where the LetsLambda is stored (not arn). This bucket must be located in your CloudFormation/Lambda region.
 - `FnPath': Path and file name to the LetsLambda package. No heading `/`
 - `Bucket`: S3 Bucket name (not arn) where the YAML configuration is located. Also used as the default location to store your certificates and priavete keys.
 - `Region`: Region short code name where the S3 bucket is located (ie: eu-west-1)
 - `ConfigFile`: Path to the YAML configuration file within the specified S3 bucket. No heading `/`
 - `KmsEncryptionKeyArn`: Default KMS Encryption Key (arn) used to securely store your SSL private keys. Use 'AES256' for S3 automatic encryption.

## Role and Managed Policies ##
As part of the deployment process, the CloudFormation template will create 4 IAM managed policies and one Lambda execution role. Each managed policy has been crafted so you can access your resources securely. The Lambda execution role defines the privilege level for the Lambda function.

 - `LetsLambdaManagedPolicy` This policy is core to the Lambda function and how it interacts with CloudWatch logs, Amazon IAM, Amazon Elastic Load Balancing and Route53.
 - `LetsLambdaKmsKeyManagedPolicy` Through this policy, the Lambda function can encrypt content when storing information into S3. Only the Lambda function should be using this role.
 - `LetsLambdaKmsKeyDecryptManagedPolicy` This policy should be used by both the Lambda function and selected EC2 instances (consumers) since it provides the mean to decrypt content stored in S3.
 - `LetsLambdaS3WriteManagedPolicy`Allow the Lambda function to write into the user defined S3 bucket.
 - `LetsLambdaS3ReadManagedPolicy` This policy is used to access any objects in the S3 bucket. Encrypted objects such as private keys will remain inaccessible until `LetsLambdaKmsKeyManagedPolicy`is used in conjunction with this policy.

### Accessing your Private keys ###
Having access to private keys is sensitive by definition. You should ensure that your private keys do not leak outside in any way.

To retrieve more easily your private keys from an EC2 instance, you should create/update an EC2 role and add both `LetsLambdaKmsKeyManagedPolicy` and `LetsLambdaS3ReadManagedPolicy`. This will allow your the EC2 instances running under the corresponding role/managed policies to access the private keys without any hard coded credentials.

## Credits ##
 - [Sébastien Requiem](https://github.com/kiddouk/)
 - [Aurélien Requiem](https://github.com/aureq/)

### Contributors ###
 - [Peter Mounce](https://github.com/petemounce)
