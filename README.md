# ionosexamples
Diverse examples how to use the IONOS APIs and command line tools

## #1 Create users and groups with ionosctl

Sometimes it may be useful to manage users and user groups on the command line. For example if you need to add multiple "technical" user accounts for your applications, using a script might be more productive than managing the accounts manually. Here are some example commands to get you started. The example adds one application user account, creates a group with S3 privileges and adds the user to the group. The App can then use the user account specific S3 secret to use the S3 services.

```bash
# Set a password for the application user account into a env variable without echoing it. Depending
# on your environment you could of course add the password here instead of asking it.
# PSWD=xxx
read -s -p"users password: " PSWD

# A simple name or "identifier" for the application (alphanumeric, no whitespace).
APP_NAME=MyApp2

# Create a new user account for the application. This will be the identity of the application. Use an email alias
# (aka. email plus addressing) so that one email account can be used for all application accounts.
ionosctl user create --first-name $APP_NAME --last-name $APP_NAME --email myemail+${APP_NAME}@example.com --password $PSWD

# get the user ID
USER_ID=$(ionosctl user list -F Firstname=$APP_NAME --no-headers --cols UserId)
echo $USER_ID

# A newly created user has no S3 privileges, we need to add the user to a group that
# has the S3 privileges. You can greate the group with the command below, but most
# would likely create the group in the data center designer with other possible privileges
# that the application group needs.
ionosctl group create --s3privilege --name MyApps

# Get the ID of the group
GROUP_ID=$(ionosctl group list  --cols Id -F Name=MyApps --cols=GroupId --no-headers )
echo $GROUP_ID

# add the application user account to the group
ionosctl group user add --group-id $GROUP_ID --user-id $USER_ID

# Get the S3 key ID of the key that was automatically generated for the user
S3_KEYID=$(ionosctl user s3key list --no-headers --cols=S3KeyId --user-id $USER_ID)
echo $S3_KEYID

# Activate the S3 key (disabled by default)
ionosctl user s3key update --s3key-active=true --s3key-id $S3_KEYID --user-id $USER_ID

# Get the secret key. the secret is not displayed in the table output. We need to resort to the json output.
S3_SECRET=$(ionosctl user s3key get --s3key-id $S3_KEYID --user-id $USER_ID --output json | jq -r ".items[0].properties.secretKey")

# Now we have the access key (S3_KEYID) and secret key (S3_SECRET) and we can
# configure the s3cmd. We also need a IONOS S3 enpoint which are listed here:
# https://docs.ionos.com/cloud/managed-services/s3-object-storage/endpoints
s3cmd --configure \
      --access_key $S3_KEYID \
      --secret_key $S3_SECRET \
      --host 's3-eu-central-1.ionoscloud.com' \
      --host-bucket '%(bucket)s.s3-eu-central-1.ionoscloud.com' \
      --region de

# Change the website endpoint, which for some reason cannot be set with the command line
# parametes above.
sed -i 's!website_endpoint.*!website_endpoint = %(bucket)s.s3-website-de-central.profitbricks.com!g' .s3cfg

# test the bucket
s3cmd mb s3://mybucket-2957894

s3cmd ls
```


## #2 Configure s3cmd for IONOS S3
s3cmd is a useful tool for troubleshooting, testing s3. You can also use it to integrate to applications to S3 with a script. Many storage applications have a native support for s3, but sometimes one needs to use a script for uploading, downloading and generally managing the data and the bucket configuration Below is an example shell session of the configuration. You will need to have the key id and secret key in the environment variables. Alternatively you can omit the "access_key" and "secret_key" from the command. The configuration process will then ask for these values.

```
~$ s3cmd --configure \
       --access_key $KEYID \
       --secret_key $KEY_SECRET \
       --host 's3-eu-central-1.ionoscloud.com' \
       --host-bucket '%(bucket)s.s3-eu-central-1.ionoscloud.com' \
       --region de

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
Access Key [xxx]:
Secret Key [xxx]:
Default Region [de]:

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
S3 Endpoint [s3-eu-central-1.ionoscloud.com]:

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3-eu-central-1.ionoscloud.com]:

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password [dummytest]:
Path to GPG program [/usr/bin/gpg]:

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
Use HTTPS protocol [Yes]:

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
HTTP Proxy server name:

New settings:
  Access Key: xxx
  Secret Key: xxxx
  Default Region: de
  S3 Endpoint: s3-eu-central-1.ionoscloud.com
  DNS-style bucket+hostname:port template for accessing a bucket: %(bucket)s.s3-eu-central-1.ionoscloud.com
  Encryption password: dummytest
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: True
  HTTP Proxy server name:
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n]
Please wait, attempting to list all buckets...
Success. Your access key and secret key worked fine :-)

Now verifying that encryption works...
Success. Encryption and decryption worked fine :-)

Save settings? [y/N] y
Configuration saved to '/home/mnylund/.s3cfg'
~$ s3cmd ls
2023-07-18 08:22  s3://mybucket-123846

```

## #3 Generate API token (IONOS_TOKEN)
You can generate a token with following curl command

```curl --user me@example.com https://api.ionos.com/auth/v1/tokens/generate```

If your email address is used for multiple contracts, you ewill need to provide the contract number as well

```curl --user me@example.com -H "X-Contract-Number: XXXXXXX"  https://api.ionos.com/auth/v1/tokens/generate```

For the ionos shell tools, the API token needs to be extracted from the returned json structure into the IONOS_TOKEN environment variable. With the following command you can extract the token from the json structure and store i into a file:

```curl --user me@example.com https://api.ionos.com/auth/v1/tokens/generate  | jq -r ".token" > .mytoken```

Alternatively, you can also use ionosctl to generate a token (you will need to have the IONOS_USERNAME and IONOS_PASSWORD environment variables exported)
```bash
ionosctl token generate
# eyJ0eXAiOiJKV1QiLC....
```

## #4 Configure the aws CLI for IONOS S3
Above I have used the s3cmd to copy data to/from the IONOS cloud, but some advanced features like object locking and SSE-C are not supported by the s3cmd. For the advanced tasks one needs to use the aws CLI. The following snippet shows how to configure the aws CLI for IONOS S3. You might also just use the aws cli for all tasks - s3cmd does not really provide anything that he aws cli would not provide. It is more like a matter of taste...
```bash
# 1. First you need to install the aws CLI. Follow instructions here:
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

# 2. Configure the aws CLI for IONOS S3. You will get the access key and secret
# access key from the data center designer GUI. replace the dummy keys below.
aws configure
# AWS Access Key ID [None]: 5cda83a9e16affddgssgsdsdfg
# AWS Secret Access Key [None]: vMfVHP6Bq3sxHrMfTkXpqSbz29Yfsdfsdfsdfsd
# Default region name [None]: de
# Default output format [None]: json

# 3. Export these variables so that you dont need to add the --endpoint-url on
# every command line. The example here use the Franfurt S3. You will need to
# refer to
# https://docs.ionos.com/cloud/managed-services/s3-object-storage/endpoints for
# other locations
export AWS_DEFAULT_REGION=de
export AWS_ENDPOINT_URL=https://s3-eu-central-1.ionoscloud.com

# 4. Add aws command line completion. This is optional but with it, using the aws cli is
# more fun. If you use the tool often, you can add the line into your .bashrc
complete -C '$(which aws_completer)' aws

# Now you are all set to use the aws CLI with IONOS S3!

# Let us create a bucket:
aws s3 mb s3://my-bucket-123763
```

more examples in the official documentation at: https://docs.ionos.com/cloud/managed-services/s3-object-storage/s3-tools/awscli

## #5 Use SSE-C encryption with aws CLI and IONOS S3

SSE-C (Server Side Encryption with Customer managed key) is a standard S3 feature which is supported by the IONOS S3 service. The following code shows how to use SSE-C with  aws CLI

```bash
# This example shows how to use SSE-C - Server Side Encryption with a Customer
# provided key. Key needs to be provided on every request.

# Lets make a test file with random data
TESTFILE=testfile.1
BUCKET_NAME=xxx-123 # add your test bucket name here (as always - needs to be domain unique)

# create a 10MB test file with random data
dd if=/dev/urandom of=$TESTFILE bs=1M count=10

# create the test bucket
aws s3 mb s3://$BUCKET_NAME

# Dummy key for the AES256 encryption. Needs to be 32 characters. Prepare also
# the other values needed for the commands
AES256_KEY="aaaaaaaabbbbbbbbccccccccdddddddd"
BASE64_ENCODED_KEY=$(echo -n $AES256_KEY|base64)
# make the md5 checksum which is needed with the 'aws s3api' low level commands
# (not needed with 'aws s3' commands)
BASE64_ENCODED_KEY_MD5=$(echo -n "$AES256_KEY" | md5sum | awk '{print $1}' | xxd -r -p | base64)
echo $AES256_KEY
echo $BASE64_ENCODED_KEY
echo $BASE64_ENCODED_KEY_MD5

# We will put the test file with the s3cmd put-object command, just to give a
# reference how the key values are used with the 'aws s3api ...' low level commands.
# Normally I would use the 'aws s3 ...' commands instead which are more convinient
# (and no need to generate md5 checksums yourself). Upload the test file:
aws s3api put-object \
        --bucket "$BUCKET_NAME" \
        --key $TESTFILE \
        --body $TESTFILE \
        --sse-customer-algorithm AES256 \
        --sse-customer-key "$BASE64_ENCODED_KEY" \
        --sse-customer-key-md5 "$BASE64_ENCODED_KEY_MD5"


# Upload a file with SSE-C encryption using the aws s3 command (more convinient)
aws s3 cp --sse-c --sse-c-key "$AES256_KEY" $TESTFILE s3://$BUCKET_NAME/${TESTFILE}.2

aws s3 ls s3://$BUCKET_NAME

# Download file with SSE-C encryption. Leawing the encryption parameters out
# would lead to a bad request error
aws s3 cp --sse-c --sse-c-key "$AES256_KEY" s3://$BUCKET_NAME/$TESTFILE ./${TESTFILE}.readback

# For the sake of end to end testing, check that the files do not differ
diff -q $TESTFILE ${TESTFILE}.readback

# remove the bucket content and the bucket itself
aws s3 rm --recursive s3://$BUCKET_NAME
aws s3 rb s3://$BUCKET_NAME

```

## #6 Enable SSE-S3 encryption with aws CLI for IONOS S3
SSE-S3 is server side encryption with S3 managed keys. Turning it on does not have notable 
performance hit (at least i could not measure any difference in upload speed). 
It ensures that data at rest is qncrypted.

```bash
# This example shows how to use the SSE-S3 encryption (Server Side Encryption
# with S3 managed keys). Using SSE-S3 is generally more convenient that the
# SSE-C since the keys do not need to be managed on the client side. But, you
# will need to trust the S3 infrastructure. Which one to use, depends solely
# on your security requirements.

TESTFILE=testfile-sse-s3
BUCKET_NAME=encryption-test-sse-s3-834832

# Lets make a test file with random data.
dd if=/dev/urandom of=$TESTFILE bs=1M count=1k

# create the test bucket
aws s3 mb s3://$BUCKET_NAME

# Turn on the SSE-S3 encryption for the bucket. This command succeeds even if
# there would be already data in the bucket. The existing data is however not
# automatically encrypted.
aws s3api put-bucket-encryption --bucket $BUCKET_NAME \
        --server-side-encryption-configuration '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'

# Check that we have the right configuration
aws s3api get-bucket-encryption --bucket $BUCKET_NAME

# Now lets copy the test file into the bucket
time aws s3 cp $TESTFILE s3://$BUCKET_NAME

aws s3api head-object --bucket $BUCKET_NAME --key $TESTFILE

# remove the test data and the bucket
aws s3 rm --recursive s3://$BUCKET_NAME
aws s3 rb s3://$BUCKET_NAME
```

## #7 Create a simple datacenter with one VM just with ionosctl
This example shows how you can create a virtual datacenter just with ionosctl in Linux shell (Bash). This is not meant to be a script that you can just run on one go (there is not error checking), but a set of exmple commands you can use to study how the virtual data center (VDC) is put togehter with ionosctl. You would normally use something like terraform for IaC, because scripts like this get quickly unmanageable (in fact it already is unamageable because there is no error checking). But, the example may however be be useful for ad-hoc testing or PoCs.

```bash
# First we set some variables to help with the commands below and create the basics in the VDC (the VC itself and the public network)
LOCATION="de/fra"
DCNAME="My VDC in $LOCATION"
IMAGE_ALIAS="ubuntu:latest"
PUBLIC_LAN_NAME=Internet
SERVERNAME="MyVm"
HOSTNAME=$SERVERNAME

# create the data center and get the datacenter ID
ionosctl datacenter create -w --name "$DCNAME" --location "$LOCATION"

# get the datacenter ID
DCID=$(ionosctl datacenter list|grep "\<$DCNAME\>"|awk '{print $1}')
echo $DCID

# Create the public LAN. You could use the grep/awk like above, but it is possible to
# select the value with the ionosctl arguments. You run into troubles however if you
# have spaces in the filter parametes (spaces in the Name for example)
ionosctl lan create --datacenter-id $DCID --public=true --name "$PUBLIC_LAN_NAME" -w
LAN_ID=$(ionosctl lan list --datacenter-id $DCID --cols LanId --no-headers --filters Name=$PUBLIC_LAN_NAME)
echo $LAN_ID

# Now, let's create a small server. With this command it will be a VM without any
# network interfaces (NICs) or volumes. They are added below.
ionosctl server create \
        --datacenter-id "$DCID" \
        --type ENTERPRISE \
        --name "$SERVERNAME" \
        --cores 1 \
        --ram $((2*1024)) \
        --cpu-family INTEL_ICELAKE \
        -w -W

# Get the server ID and store in into a variable
SERVERID=$(ionosctl server list --datacenter-id $DCID | grep "$SERVERNAME" | awk '{print $1}')
echo $SERVERID

# Create a NIC and get the public IP address after creation.
ionosctl nic create \
        --datacenter-id $DCID \
        --server-id $SERVERID \
        --name "MyInternetNIC" \
        --lan-id "$LANID" \
        -w
SERVERIP=$(ionosctl nic list --datacenter-id $DCID --server-id $SERVERID | grep Internet | awk '{print $5}')
echo $SERVERIP

# Now, if you look into the the data center designer, you will find a VM
# connected to internet. But it has not boot volume yet. Let's create the
# volume and attach it to the VM. We will add a small cloud-init to the image
# which will set the VMs hostname. You will need to set the root password
# variable to some value. We also include our ssh key (public key) to the image
# so that we can directly ssh into it.
VOLNAME=$HOSTNAME
CLOUDINIT=$(echo -e "#cloud-config\n\nhostname: $HOSTNAME\n" | base64 -w0)
# Create the boot volume
ionosctl volume create \
        --datacenter-id "$DCID" \
        --name "$VOLNAME"\
        --image-alias $IMAGE_ALIAS \
        --user-data "$CLOUDINIT" \
        --type "SSD Premium" --size 300GB \
        --ssh-key-paths "$HOME/.ssh/id_rsa.pub" \
        --password $ROOT_PASSWORD \
        -w

# We need to get the volume id so that we can attach the volume to the server
VOLUMEID=$(ionosctl volume list --datacenter-id $DCID | grep "$VOLNAME" | awk '{print $1}')
echo "VolumeID: $VOLUMEID"

# Now attach the newly created volume to the server
ionosctl server volume attach \
        --datacenter-id $DCID \
        --server-id $SERVERID \
        --volume-id $VOLUMEID \
        -w

# Now the server is up and running and we can ssh into it.
ssh root@$SERVERIP

```
