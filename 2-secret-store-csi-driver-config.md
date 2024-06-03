# Connect your Azure ID to the Azure Key Vault Secrets Store CSI Driver

### Configure workload identity
```
$SUBSCRIPTION_ID = "fe4a1fdb-6a1c-4a6d-a6b0-dbb12f6a00f8"
$RESOURCE_GROUP = "keyvault-demo"
$UAMI = "azurekeyvaultsecretsprovider-keyvault-demo-cluster"
$KEYVAULT_NAME = "aks-demo-abhi"
$CLUSTER_NAME = "keyvault-demo-cluster"
```
### Set subscription
```
az account set --subscription $SUBSCRIPTION_ID

```
### Create a managed identity
```
az identity create --name $UAMI --resource-group $RESOURCE_GROUP | ConvertFrom-Json
$USER_ASSIGNED_CLIENT_ID = (az identity show -g $RESOURCE_GROUP --name $UAMI --query clientId -o tsv)
```

### Get identity tenant
```
$IDENTITY_TENANT = (az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)
```
### Create a role assignment
```
$KEYVAULT_SCOPE = (az keyvault show --name $KEYVAULT_NAME --query id -o tsv)
az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```
### Get AKS OIDC Issuer URL
```
$AKS_OIDC_ISSUER = (az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)
```
### Create service account for the pod
```
$SERVICE_ACCOUNT_NAME = "workload-identity-sa"
$SERVICE_ACCOUNT_NAMESPACE = "default"
@"
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: $USER_ASSIGNED_CLIENT_ID
  name: $SERVICE_ACCOUNT_NAME
  namespace: $SERVICE_ACCOUNT_NAMESPACE
"@ | kubectl apply -f -

# Setup Federation
$FEDERATED_IDENTITY_NAME = "aksfederatedidentity"
az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer $AKS_OIDC_ISSUER --subject "system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}" | ConvertFrom-Json

# Create the Secret Provider Class
@"
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "$USER_ASSIGNED_CLIENT_ID" # Setting this to use workload identity
    keyvaultName: $KEYVAULT_NAME       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: secret1             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: key1                # Set to the name of your key
          objectType: key
          objectVersion: ""
    tenantId: "$IDENTITY_TENANT"        # The tenant ID of the key vault
"@ | kubectl apply -f -
```
