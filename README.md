# Salesforce DX CI/CD with GitHub Actions and Azure Key Vault

This simple scenario shows how to leverage GitHub actions for adopting a CI/CD process for Salesforce development and it is heavily dependant on the **Prerequisites** in order for the continious integration and continious deployment. The GitHub workflow shows how to succcessfully provision the **SFDX CLI** to login and validate connection to the Salesforce org, which implicitaly allows you to use **force:source** topic to pull, and push your app as well as the **force:apex** topic for testing your app and the full range of topics and commands provided by the CLI.

## Prerequisites

1. [Develop an app with Salesforce CLI and source control](https://trailhead.salesforce.com/content/learn/projects/develop-app-with-salesforce-cli-and-source-control)
2. [Create your connected app](https://trailhead.salesforce.com/content/learn/modules/sfdx_travis_ci/sfdx_travis_ci_connected_app?trail_id=sfdx_get_started)

## Setup environment

### Azure KeyVault

```bash
# Set env vars
resourceGroup="<RG-NAME>"
location="<LOCATION>"
keyVaultName="<KV_NAME>"

# Create resource group
az group create \
    -n $resourceGroup \
    -l $location

# Create KV
az keyvault create \
    -n $keyVaultName\
    -g $resourceGroup
    -l $location
```

### Create PFX certificate and import into Azure KeyVault

```bash
# Vars
appCert="<APPCERT-NAME>.pfx"
certKey="<KEY-NAME>.key"
cert="<CERT-NAME>.crt"
keyVaultCertName="<KV-CERT-NAME>"
certPassphrase="<PASSPHRASE>"

# Generate PFX from certificate and key created from the pre-requisites
openssl pkcs12 -export \
    -out $appCert \
    -inkey $certKey \
    -in $cert

# Import PFX into KV
az keyvault certificate import \
    --vault-name $keyVaultName \
    -n $keyVaultCertName \
    -f $appCert \
    --password $certPassphrase
```

### Configure GitHub secrets

```bash
# Consumer Key or Client ID from the Salesforce portal 
SFDX_CONSUMER_KEY

# Username
SFDX_HUB_USERNAME

# Key file - we may not need this as it's generated ????
SFDX_JWT_KEY

# Azure credentials - output of az ad sp create-for-rbac 
SFDX_AZ_CREDS
```

## GitHub Actions for SFDX

### Get certificate from Azure Key Vault

Salesforce DX CLI uses JWT Auth flow and the CLI requires the path and name for the private key to be passed as an argument, **Get cert** step does the following:

1. Creates a directory for storing the cert
2. Uses the Azure CLI to download the secret
3. Decodes the secret into another file
4. Extracts the RSA key

```yaml
      - name: Get cert
        run: |
          echo "Base path: " $(pwd)
          mkdir ${{ env.SFDX_KEY_DIR }} && cd ${{ env.SFDX_KEY_DIR }}
          echo "SFDX key path: " $(pwd)
          az keyvault secret download --vault-name ${{ env.KEY_VAULT }} -n ${{ env.KV_CERT }} -f ${{ env.CERT }}
          cat ${{ env.CERT }} | base64 -d > ${{ env.DECODED_CERT }}
          openssl pkcs12 -in ${{ env.DECODED_CERT }} -nocerts -nodes -out ${{ env.DECODED_PEM }} -passin pass:
          openssl rsa -in ${{ env.DECODED_PEM }} -out ${{ env.DECODED_KEY }}
```

### Preparing Salesforce DX environment

Salesforce DX uses its CLI and **force** topic to provide tools for developers. The **Get SFDX CLI** downloads and installs the CLI directly from Salesforce.

```yaml
      - name: Get SFDX CLI
        run: |
          echo "Base path: " $(pwd)
          wget https://developer.salesforce.com/media/salesforce-cli/sfdx-linux-amd64.tar.xz
          mkdir ${{ env.SFDX_CLI_DIR }}
          tar xJf sfdx-linux-amd64.tar.xz -C ${{ env.SFDX_CLI_DIR }} --strip-components 1
          ./${{ env.SFDX_CLI_DIR }}/install
          sfdx --version
```

### Salesforce DX operations

The following two steps perform the login and issues list and display to validate connection is successful

```yaml
      - name: SFDX Login
        run: |
          sfdx force:auth:jwt:grant --clientid ${{ secrets.SFDX_CONSUMER_KEY }} --jwtkeyfile ${{ env.SFDX_KEY_DIR }}/${{ env.DECODED_KEY }} --username ${{ secrets.SFDX_HUB_USERNAME }} --setdefaultdevhubusername
          
      - name: SFDX Display and List
        run: |
          sfdx force:org:list
          sfdx force:org:display
```
