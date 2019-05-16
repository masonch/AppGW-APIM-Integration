# Deploy an Application Gateway V2 in front of an internal API Management instance

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmasonch%2FAppGW-APIM-Integration%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

## Overview

This quickstart will deploy an API Management instance in *internal* mode to a VNET. In order to allow it to have controlled external access, an Application Gateway will be deployed in front. For both API Management and Application Gateway, the appropriate certificates used for the custom domains are being pulled from Key Vault using managed identities.

In order to deploy this template, you need to have the following resources already deployed:

1. A Key Vault.
2. A certificate in PFX format uploaded to the Key Vault for the custom domain you want to use for template. (See PowerShell script below)
3. The public key to the trusted root authority for the certificate of the custom domain. (See PowerShell script below)
4. Enable the Key Vault for [ARM deployment][keyvault-enable-deployment].
5. Enable the Key Vault for [soft delete][keyvault-soft-delete].

## PowerShell Scripts

```powershell
# Upload PFX to Key Vault
$pfxFilePath = "PFX_CERTIFICATE_FILE_PATH" # Change this path
$pwd = "PFX_CERTIFICATE_PASSWORD"  # Change this password
$vaultName = "VAULT_NAME"
$secretName = "KEY_VAULT_SECRET_NAME"

$flag = [System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable
$collection = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2Collection
$collection.Import($pfxFilePath, $pwd, $flag)
$pkcs12ContentType = [System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12
$clearBytes = $collection.Export($pkcs12ContentType)
$fileContentEncoded = [System.Convert]::ToBase64String($clearBytes)
$secret = ConvertTo-SecureString -String $fileContentEncoded -AsPlainText –Force
$secretContentType = 'application/x-pkcs12'
Set-AzureKeyVaultSecret -VaultName $vaultName -Name $secretName -SecretValue $Secret -ContentType $secretContentType # Change Key Vault name and Secret name
```

```powershell
# Find trusted root public certificate
$cerFile = "CER_CERTIFICATE_FILE_PATH"

$certificate = New-Object Security.Cryptography.X509Certificates.X509Certificate2
$certificate.Import($cerFile)

$chain = New-Object Security.Cryptography.X509Certificates.X509Chain
$chain.Build($certificate)
$rootCert = $chain.ChainElements[$chain.ChainElements.Count - 1].Certificate
$base64 = [System.Convert]::ToBase64String($rootCert.RawData)
$chain.Reset()

$base64
```

```powershell
# Enable Key Vault for soft delete
$vaultName = "VAULT_NAME"

($resource = Get-AzResource -ResourceId (Get-AzKeyVault -VaultName $vaultName).ResourceId).Properties | Add-Member -MemberType "NoteProperty" -Name "enableSoftDelete" -Value "true"

Set-AzResource -resourceid $resource.ResourceId -Properties $resource.Properties
```

## Deployment

A summary of the parameters required for this template are in this table.

|Parameter|Details|
|-|-|
|location|Location to deploy the resources|
|vnetResourceName|Name of the virtual network|
|vnetAddressRange|Range of the addresses for the virtual network|
|apiManagementSubnetName|Name of the subnet to hold the API Management resource|
|apiManagementSubnetRange|Range of addresses for the API Management instance in the virtual network|
|appGWSubnetName|Name of the subnet to hold the Application Gateway resource|
|appGWSubnetRange|Range of addresses for the Application Gateway instance in the virtual network|
|appGWResourceName|Name of the Application Gateway|
|appGWPublicDNSPrefix|DNS prefix to use for the Application Gateway|
|apiManagementResourceName|Name of the API Management instance|
|apiManagementGatewayCustomDomain|Custom domain for the gateway / proxy address in API Management|
|apiManagementPortalCustomDomain|Custom domain for the developer portal address in API Management|
|apiManagementPublisherEmail|Email address of the publisher for API Management|
|apiManagementPublisherName|Name of the publisher for API Management|
|apiManagementSku|The SKU to use for API Management. Since this example is putting API Management in a virtual network, the options allowed are *Developer* or *Premium*|
|apiManagementSkuCount|Size of the API Management instance|
|existingKeyVaultName|Name of the Key Vault that holds the PFX certificate|
|existingKeyVaultSecret|Name of the secret of the PFX certificate|
|trustedRootBase64PublicKey|Base-64 encoding of the trusted root certificate|
|tags|Any tags to use in the deployment|

Once the resource is deployed, you should be able to point your DNS to the CNAME record of the deployed Application Gateway and connect to API Management.

<!--Links -->
[keyvault-enable-deployment]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-keyvault-parameter#deploy-key-vaults-and-secrets
[keyvault-soft-delete]: https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-powershell#enabling-soft-delete