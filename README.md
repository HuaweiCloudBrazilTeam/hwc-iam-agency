# Using an IAM Agency to access an OBS bucket from an ECS instance

The several attempts to access an OBS bucket from an ECS instance, using an IAM Agency, are documented here.

These tests were made in an ECS instance with *Ubuntu 18.04*, in *ap-southeast-1* region.

```bash
export OBS_HOST=obs.ap-southeast-1.myhuaweicloud.com
```

The instance had an IAM Agency attached to it, with the following roles attached to the IAM Agency:
* OBS OperateAccess (theoretically just this one is needed)
* Tenant Administrator (useful for doing other API operation attempts)

## Extracing the AK/SK/Token values and storing them in environment variables

`jq` is helpful to extract values from the json output from the metadata API 

```bash
apt update && apt install jq -y 

## Viewing the metadata API output, formatted with jq
curl -s http://169.254.169.254/openstack/latest/securitykey | jq
```

### These values expire after 15 minutes, so you need to constantly rerun this

The AK/SK/Token values are changed for every new requrest to the metadata API (http://169.54.169.254), and they need to be used as a set. **This means that AK/SK/Token must be extract from the same API request, and not separate ones.**

```bash
export IAM_AGENCY_SECURITY_KEY=$(curl -s http://169.254.169.254/openstack/latest/securitykey | jq .)
export IAM_AGENCY_AK=$(echo $IAM_AGENCY_SECURITY_KEY | jq -r .credential.access)
export IAM_AGENCY_SK=$(echo $IAM_AGENCY_SECURITY_KEY | jq -r .credential.secret)
export IAM_AGENCY_TOKEN=$(echo $IAM_AGENCY_SECURITY_KEY | jq -r .credential.securitytoken)
```


## Using `obsutil`

* [OBS installation instructions](https://support.huaweicloud.com/intl/en-us/utiltg-obs/obs_11_0003.html)

```bash
wget https://obs-community-intl.obs.ap-southeast-1.myhuaweicloud.com/obsutil/current/obsutil_linux_amd64.tar.gz
tar xzf obsutil_linux_amd64.tar.gz
cd obsutil_linux_amd64_5.1.13
chmod +x obsutil
cp obsutil /usr/local/bin
obsutil version
```

### `obsutil` refuses to work without an `.obsutilconfig` (even when all parameters are passed through the CLI)
This will create an empty configuration file if not already existant. 
If it already exists, will keep the file as it is.

```bash
 touch -a ~/.obsutilconfig
```

### Calling `obsutil` passing the Securitykey parameters through CLI

```bash
obsutil ls \
   -e=$OBS_HOST \
   -i=$IAM_AGENCY_AK \
   -k=$IAM_AGENCY_SK \
   -t=$IAM_AGENCY_TOKEN
```

### Configuring `obsutil` with the AK/SK and Token

* Don't forget that the security key expires! You will need to eventually refresh it.
* `-t` flag is undocumented

```bash
obsutil config \
   -e=$OBS_HOST \
   -i=$IAM_AGENCY_AK \
   -k=$IAM_AGENCY_SK \
   -t=$IAM_AGENCY_TOKEN

obsutil ls
```

##  OBS权限配置实践--委托服务进行OBS访问 
[This is a tutorial](https://bbs.huaweicloud.com/blogs/100733) on how to access an OBS bucket, from an ECS instance, using the IAM Agency, using S3curl as the S3 client.

**It's exactly what we need, but it's not working.**

```bash
apt install s3curl -y
```

I made an adaption, avoiding the need to constantly edit the `.s3curl` configuration file (as the temporary credentials expires in a few minutes).

```bash
s3curl \
--id $IAM_AGENCY_AK \
--key $IAM_AGENCY_SK \
-- \
-H "x-amz-security-token: $IAM_AGENCY_TOKEN" \
https://obs-for-agency-access.obs.ap-southeast-1.myhuaweicloud.com/test123.txt \
2>/dev/null
```

