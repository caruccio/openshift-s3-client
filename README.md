# OpenShift S3 Client (S3C)

This cartridge ask a central authority for S3 credentials, exporting thoses credentials as environment variables.

This is a first initial testing relase, so do no even try to use it.

I'm putting it here expeting some feedback and ideas on how to implement in a secure way.
Please, contributions are very welcome.

## Architecture

There are 2 components:

### S3 Client (this cartridge)

Requests a credential server for S3 credentials.
Those credentials will be generated on-the-fly.

After created, they are published to all cartridges as environment variables:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_BUCKET_NAME

### Credential Server (CS)

- Receive the credentials request and authenticate.
- Asks AWS for IAM credentials (under provider's account).
- Register cartridge usage

Credential Server API exposes a RESTful interface as follow:

```
https://<server-addr>/<provider>/<service>
```

For example, to request a AWS S3 credential the request URL should be:

```
https:/<server-addr>/aws/s3
```

### Authentication

Upon installation, S3C creates a file inside the main app gear containing a random generated salt, used to sign the request.
This file is located at `$OPENSHIFT_S3_CLIENT_DIR/.salt`.

The request itself is a plain text POST containing some environment variables from gear and a hash of the entire message.
The format of the message is as follow:

```
$OPENSHIFT_APP_UUID|$OPENSHIFT_APP_DNS|$OPENSHIFT_S3_CLIENT_DIR|<MESSAGE-HASH>
```

For example, given following salt value `SALT=d72ebd92de8230e4`, one can generate <MESSAGE-HASH> with:

```
echo "$OPENSHIFT_APP_UUID|$OPENSHIFT_APP_DNS|$OPENSHIFT_S3_CLIENT_DIR|$SALT" | sha256sum
8b7b830cd1ec536a5f681947c3b28857d63845fcc07eb0ddf57d12fe2ce20137 -
```

The message will looks like this:

```
52b5bfbda665b7779400004f|myapp-example.getup.io|/var/lib/openshift/52b5bfbda665b7779400004f/|8b7b830cd1ec536a5f681947c3b28857d63845fcc07eb0ddf57d12fe2ce20137
```

After reading the message, CS then tries to authenticate it. First, it reads the salt file inside the gear (content from $OPENSHIFT_S3_CLIENT_DIR/.salt).
With that value, it then re-generate the signature using the values from request. If values match than it is authenticated an processing continues. If not, the connection is dropped with a proper error message.

### Credentials creation

Each provider has a different way to create credentials. For instance, we will start implementing only AWS S3 credentials, which uses a service called [IAM](http://aws.amazon.com/iam/).
This service allows for a main account to create different credentials for different services provided by AWS. There will be a map of each user application to a distinct AWS account.

To create a new user credential:

- Using vendor's credentials, request a new IAM user and access key.
- Attach restrictive policy to new user, allowing S3 access only without bucket delete permission.
- Register this new resource for billing (maybe into mongodb usage_record?)
- Return username, access key and bucket name.

Example routine to create IAM user and key:

```python
from boto import iam
from datetime import datetime
from json import dumps

def create_aim_user(vendor_key_id, vendor_access_key, openshift_app_uuid):
	client = iam.connection.IAMConnection(vendor_key_id, vendor_access_key)
	user   = c.create_user(openshift_app_uuid)
	key    = c.create_access_key(openshift_app_uuid)
	policy = {
		"Statement": [
			{
				"Sid": "Stmt{0}-{1}".format(openshift_app_uuid, datetime.strftime(datetime.utcnow(), '%s')),
				"Effect": "Allow",
				"Action": [
					"s3:AbortMultipartUpload",
					"s3:DeleteBucketPolicy",
					"s3:DeleteBucketWebsite",
					"s3:DeleteObject",
					"s3:DeleteObjectVersion",
					"s3:GetBucketAcl",
					"s3:GetBucketLocation",
					"s3:GetBucketLogging",
					"s3:GetBucketNotification",
					"s3:GetBucketPolicy",
					"s3:GetBucketRequestPayment",
					"s3:GetBucketTagging",
					"s3:GetBucketVersioning",
					"s3:GetBucketWebsite",
					"s3:GetLifecycleConfiguration",
					"s3:GetObject",
					"s3:GetObjectAcl",
					"s3:GetObjectTorrent",
					"s3:GetObjectVersion",
					"s3:GetObjectVersionAcl",
					"s3:GetObjectVersionTorrent",
					"s3:ListBucket",
					"s3:ListBucketMultipartUploads",
					"s3:ListBucketVersions",
					"s3:ListMultipartUploadParts",
					"s3:PutBucketAcl",
					"s3:PutBucketLogging",
					"s3:PutBucketNotification",
					"s3:PutBucketPolicy",
					"s3:PutBucketRequestPayment",
					"s3:PutBucketTagging",
					"s3:PutBucketVersioning",
					"s3:PutBucketWebsite",
					"s3:PutLifecycleConfiguration",
					"s3:PutObject",
					"s3:PutObjectAcl",
					"s3:PutObjectVersionAcl"
				],
				"Resource": [
					"arn:aws:s3:::{0}/*".format(openshift_app_uuid)
				]
			}
		]
	}
	client.put_user_policy(openshift_app_uuid, 'default', dumps(policy, indent=3))
```
