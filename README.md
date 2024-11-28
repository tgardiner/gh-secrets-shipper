# gh-secrets-shipper
[gh-secrets-shipper](gh-secrets-shipper.yml) is a Github Actions Workflow that encrypts the repository's secrets and uploads them to an S3 bucket.

## Requirements
It requires the following secrets to be configured on the repository:
```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
KMS_KEY_ID
S3_SECRETS_BUCKET
```

## Decrypting the secrets
A disaster has happened, someone has deleted the secrets from the Github repository, or maybe Github just lost them :fearful:

**Don't worry** - You have multiple copies of the secrets stored in S3, with bucket versioning enabled!

### Download the encrypted secrets & datakey
```
aws s3 cp s3://<my-bucket>/<my-repo>/secrets .
aws s3 cp s3://<my-bucket>/<my-repo>/secrets.key .
```

### Decrypt the datakey with KMS
```
datakey=$(aws kms decrypt --key-id <my-keyid> --ciphertext-blob $(cat secrets.key) --query Plaintext --output text)
```

### Decrypt the secrets with GPG
```
gpg --batch --passphrase $datakey --no-symkey-cache --decrypt <(cat secrets | base64 -d) | jq .
{
  "AWS_REGION": "us-east-1",
  "KMS_KEY_ID": "11111111-1111-1111-1111-111111111111",
  "AWS_ACCESS_KEY_ID": "AAAAAAAAAAAAAAAAAAAA",
  "S3_REPO_BUCKET": "gh-secrets-shipper",
  "AWS_SECRET_ACCESS_KEY": "XXXXXXXXXXXXXXXXXXXXX"
}
```

## AWS Setup
You'll need to create:
* An S3 bucket
* A KMS key
* An IAM user with access to upload to the bucket

### Create the S3 bucket
```
aws s3 mb s3://<my-bucket>
```

#### Enable bucket versioning
```
aws s3api put-bucket-versioning \
  --bucket <my-bucket> \
  --versioning-configuration Status=Enabled
```

#### Enable bucket encryption
```
aws s3api put-bucket-encryption \
  --bucket <my-bucket> \
  --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'
```

### Create the KMS key
```
aws kms create-key
```

> {  
    "KeyMetadata": {  
        "AWSAccountId": "111111111111",  
        "KeyId": "11111111-1111-1111-1111-111111111111",  
        "Arn": "arn:aws:kms:us-east-1:111111111111:key/11111111-1111-1111-1111-111111111111",  
        "CreationDate": "2024-10-18T00:00:00.000000-00:00",  
        ...

#### Create a key alias
```
aws kms create-alias --target-key-id <my-keyid> --alias-name alias/gh-secrets-shipper
```

#### Allow all IAM users to encrypt
```
aws kms put-key-policy --key-id <my-keyid> --policy file://key_policy.json
```

### Create the IAM user
```
aws iam create-user \
  --user-name <my-user>
```

#### Create a policy that allows putting the secrets to s3
```
aws iam create-policy \
    --policy-name <my-policy> \
    --policy-document \
'{
  "Version" : "2012-10-17",
  "Statement" : [
    {
      "Effect" : "Allow",
      "Action" : [
        "s3:PutObject"
      ],
      "Resource" : [
        "arn:aws:s3:::<my-bucket>/<my-repository>/secrets.key",
        "arn:aws:s3:::<my-bucket>/<my-repository>/secrets",
      ]
    }
  ]
}'
```

#### Assign the policy to the user
```
aws iam attach-user-policy \
    --policy-arn arn:aws:iam::<my-account>:policy/<my-policy> \
    --user-name <my-user>
```
