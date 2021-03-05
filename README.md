

# combine cert and key
openssl pkcs12 -export -out certificate.pfx -inkey key.pem -in cert.pem

# Import into KV
az keyvault certificate import --vault-name mykeyvault -n mycert -f certificate.pfx --password "<password>"

# download cert
az keyvault certificate download --vault-name mykeyvault -n mycert -f downloadedcert.pem

$ az keyvault secret download --vault-name mykeyvault -n mycert --file downloaded.pfx

# Decode from Base64
$ cat downloaded.pfx | base64 -d > decoded.pfx

# Verify we can read the file
$ openssl pkcs12 -in decoded.pfx

# Extract key
openssl pkcs12 -in certname.pfx -nocerts -out key.pem -nodes
openssl rsa -in key.pem -out server.key 

################################################################

# Salesforce DX CI/CD with GitHub Actions and Azure KeyVault

### Prerequisites

1. [Develop an app with Salesforce CLI and source control](https://trailhead.salesforce.com/content/learn/projects/develop-app-with-salesforce-cli-and-source-control)
2. [Create your connected app](https://trailhead.salesforce.com/content/learn/modules/sfdx_travis_ci/sfdx_travis_ci_connected_app?trail_id=sfdx_get_started)

### Azure KeyVault

```bash
# Create resource group
az group create --name "myResourceGroup" -l "EastUS"

# Create KV
az keyvault create --name "<your-unique-keyvault-name>" --resource-group "myResourceGroup" --location "EastUS"
```

### Create PFX certificate and import into Azure KeyVault

```bash
# Generate PFX from certificate and key created
openssl pkcs12 -export -out certificate.pfx -inkey server.key -in server.crt

# Import PFX into KV
az keyvault certificate import --vault-name mykeyvault -n mycert -f certificate.pfx --password "<password>"

```

### Configure GitHub secrets

```bash
# Consumer Key or Client ID from the Salesforce portal 
SFDX_CONSUMER_KEY

# Username
SFDX_HUB_USERNAME

# Key file - we may not need this as it's generated ????
SFDX_JWT_KEY

# Azure credentials
SFDX_AZ_CREDS
```

10 vms - 64gb - 16cores - build agents 1 core min 
team city - octupus deploy
10 - 15 