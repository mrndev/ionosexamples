# ionosexamples
Diverse examples how to use the IONOS APIs and command line tools

## #1 Create users and groups with ionosctl

Sometimes it may be useful to manage users and user groups on the command line. For example if you need to add multiple "technical" user accounts for your applications, using a script might be more productive than managing the accounts manually. Here are some example commands to get you started. The example adds one application user account, creates a group with S3 privileges and adds the user to the group. The App can then use the user account specific S3 secret to use the S3 services.

```
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


