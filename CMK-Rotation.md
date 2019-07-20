# Manual CMK Demo
***

# Create CMK with No Key Material
## Inputs
* `$CMK_ALIAS`: Friendly name for Customer Master Key (CMK)
* `$CMK_REGION`: CMK-Owning Region 
* `$CMK_OWNER`: CMK-Owning AWS Account Id
* `$CMK_ADMIN_ROLE1`: CMK Administration Role
* `$CMK_USER_ACCOUNT1`: CMK-Using AWS Account Id
* `$CMK_USER_ROLE1`: Role in CMK-Using account with permission to use CMK

## Steps
* Login to account `$CMK_OWNER` as `$CMK_ADMIN_ROLE1`
* Service -> KMS -> Customer managed keys
* Select Region: `$CMK_REGION`
* Select: Create Key
* Enter Alias: `$CMK_ALIAS`
* Select Advanced Options
* Choose: External
* Select CMK Key Administrators from list of IAM users and roles in CMK-owning account: `$CMK_ADMIN_ROLE1`
* Select other AWS accounts that can use the CMK: `$CMK_USER_ACCOUNT1`

## Outputs
* `$CMK_ID1`: CMK Identifier
* `$CMK_ID_ARN1`: arn:aws:kms:`$CMK_REGION`:`$CMK_OWNER`:key/`$CMK_ID1`
* `$CMK_ALIAS_ARN`: arn:aws:kms:`$CMK_REGION`:`$CMK_OWNER`:alias/`$CMK_ALIAS`

`$CMK_POLICY_TEXT`
```json
{
    "Id": "key-consolepolicy-1",
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$CMK_OWNER:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "Allow access for Key Administrators",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$CMK_OWNER:role/$CMK_ADMIN_ROLE1"
            },
            "Action": [
                "kms:Create*",
                "kms:Describe*",
                "kms:Enable*",
                "kms:List*",
                "kms:Put*",
                "kms:Update*",
                "kms:Revoke*",
                "kms:Disable*",
                "kms:Get*",
                "kms:Delete*",
                "kms:ImportKeyMaterial",
                "kms:TagResource",
                "kms:UntagResource",
                "kms:ScheduleKeyDeletion",
                "kms:CancelKeyDeletion"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Allow use of the key",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::CMK_USER_ACCOUNT1:root"
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Allow attachment of persistent resources",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::CMK_USER_ACCOUNT1:root"
            },
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        }
    ]
}
```

# Download Public Key and Token for Importing Key Material

## Inputs
* `$MYDIR`: Local working directory for generating key material

## Steps
* Service -> KMS -> Customer managed keys
* Select: `$CMK_ALIAS`
* Click 'Download wrapping key and import token' button
* Select wrapping algorithm: RSAES_OAEP_SHA_1
* Click: Download
* Save file: ImportParameters.zip
* Unzip ImportParameters.zip into `$MYDIR`

## Outputs
* File: $MYDIR/wrappingKey_$CMK_ID1_$TIMESTAMP
* File: $MYDIR/importToken_$CMK_ID1_$TIMESTAMP
* File: $MYDIR/README_$CMK_ID1_$TIMESTAMP.txt

# Generate and Encrypt Key Material

## Steps
* Open Linux/Windows command line
* cd $MYDIR
* rename wrappingKey_$CMK_ID1_$TIMESTAMP to PublicKey1.bin
* rename importToken_$CMK_ID1_$TIMESTAMP to ImportToken1.bin
* Generate a 256-bit symmetric key by entering the following command:
```
openssl rand -out PlaintextKeyMaterial1.bin 32
```

* Encrypt the plain text key material using the RSA public key
```
openssl rsautl -pubin -encrypt -oaep -keyform DER \
 -in PlaintextKeyMaterial1.bin \
 -inkey PublicKey1.bin \
 -out EncryptedKeyMaterial1.bin
```

# Import Encrypted Key Material into CMK

## Steps
* Service -> KMS -> Customer managed keys
* Select: `$CMK_ALIAS`
* Click 'Upload key material' button
* Click 'Wrapped key material' button
* Choose file: $MYDIR/EncryptedKeyMaterial1.bin
* Click 'Import token' button
* Choose file: $MYDIR/ImportToken1.bin
* Click 'Upload key material' button
* Check that status of `$CMK_ALIAS` has changed from Pending to Enabled


# Use CMK to Encrypt S3 objects in an Application Account

## Inputs
* `$MY_ENCRYPTED_BUCKET`: A sandbox bucket for testing key rotation
* `$MY_SAMPLE_OBJECT1`: A sample object for testing key rotation

## Steps
* Login to Application Account `$CMK_USER_ACCOUNT1` as `$CMK_USER_ROLE1`
* Service -> S3
* Click: Create Bucket
* Enter Name: `$MY_ENCRYPTED_BUCKET`
* Select Region: `$CMK_REGION`
* Select Default Encryption:
  * AWS-KMS
  * Custom KMS ARN: `$CMK_ALIAS_ARN`
  ***
* Upload `$MY_SAMPLE_OBJECT1` into `$MY_ENCRYPTED_BUCKET`, accepting the default encryption option (labelled None) - which is AWS-KMS with `$CMK_ALIAS_ARN`.


# Rotate CMK

* Open Linux/Windows command line
* cd $MYDIR
* Set CLI temporary credentials for `$CMK_OWNER` as `$CMK_ADMIN_ROLE1`
* Create new CMK with No Key Material, and assign id to `$CMK_ID2`. 
```
CMK_ID2=$(aws kms create-key --origin EXTERNAL --query 'KeyMetadata.KeyId' --output text)
```
* Generate a public wrapping key and import token for `$CMK_ID2`.
```
aws kms get-parameters-for-import --key-id $CMK_ID2 \
                                    --wrapping-algorithm RSAES_OAEP_SHA_1 \
                                    --wrapping-key-spec RSA_2048
```
* Locate the attributes __PublicKey__ and __ImportToken__ in the JSON output of the command.
* Save the string value of the __PublicKey__ attribute in a file called __PublicKey2.b64__.
* Save the string value of the __ImportToken__ attribute in a file called __ImportToken2.b64__.
* Convert the public key and token from base64 to binary
```
openssl enc -d -base64 -A -in PublicKey2.b64 -out PublicKey2.bin
openssl enc -d -base64 -A -in ImportToken2.b64 -out ImportToken2.bin
```
* Generate key material for `$CMK_ID2` and save in a binary file called __PlaintextKeyMaterial2.bin__.
```
openssl rand -out PlaintextKeyMaterial2.bin 32
```
* Encrypt the generated key material __PlaintextKeyMaterial2.bin__ using __PublicKey2__ and save the output to a file called __EncryptedKeyMaterial2.bin__.
```
openssl rsautl -pubin -encrypt -oaep -keyform DER \
  -in PlaintextKeyMaterial2.bin \
  -inkey PublicKey2.bin \
  -out EncryptedKeyMaterial2.bin
```
* Import the encrypted key material for `$CMK_ID2` from the file __EncryptedKeyMaterial2.bin__, passing back the (binary) import token contained in __ImportToken2.bin__.
```
aws kms import-key-material --key-id $CMK_ID2 \
                              --encrypted-key-material fileb://EncryptedKeyMaterial2.bin \
                              --import-token fileb://ImportToken2.bin \
                              --expiration-model KEY_MATERIAL_DOES_NOT_EXPIRE
```
* Download the key policy for `$CMK_ID1` and save in a json file called __PolicyText.json__.
```
aws kms get-key-policy --key-id $CMK_ID1 --policy-name default --output text > PolicyText.json
```
* Attach the key policy in __PolicyText.json__ to `$CMK_ID2`
```
aws kms put-key-policy --key-id $CMK_ID2 --policy-name default --policy file://PolicyText.json
```
* Redirect the key alias `$CMK_ALIAS` from `$CMK_ID1` to `$CMK_ID2`
```
aws kms update-alias --alias-name alias/$CMK_ALIAS --target-key-id $CMK_ID2
```

# Use the rotated CMK to Encrypt S3 objects in an Application Account

## Inputs
* `$MY_SAMPLE_OBJECT2`: A sample object for testing key rotation

## Steps
* Login to Application Account `$CMK_USER_ACCOUNT1` as `$CMK_USER_ROLE1`
* Service -> S3
* Open bucket `$MY_ENCRYPTED_BUCKET`
* Upload `$MY_SAMPLE_OBJECT2` into `$MY_ENCRYPTED_BUCKET`, accepting the default encryption option (labelled None) - which is AWS-KMS with `$CMK_ALIAS_ARN`.
* Confirm that upload is successful

# Confirm that objects encrypted prior to CMK rotation remain accessible

## Steps
* Login to Application Account `$CMK_USER_ACCOUNT1` as `$CMK_USER_ROLE1`
* Service -> S3
* Open bucket `$MY_ENCRYPTED_BUCKET`
* Download `$MY_SAMPLE_OBJECT1`. This object was encrypted using a data key that can only be decrypted by `CMK_ID1` material.
* Download `$MY_SAMPLE_OBJECT2`. This object was encrypted using a data key that can only be decrypted by `CMK_ID2` material.

****