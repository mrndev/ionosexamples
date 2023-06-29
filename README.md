# ionosexamples
Diverse examples how to use the IONOS APIs and command line tools

## Create users and groups with ionosctl

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
