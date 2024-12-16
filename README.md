# Multi-Cluster AKS Management using Flux and ASO
This repo provides a sample implementation using [Flux]() and [Azure Service Operator]() to manage a dynamic fleet of Azure Kubernetes Clusters.

## TODO: Architecture

## Build the Control Plane Cluster
In order to bootstrap the fleet we need our first ASO cluster. This cluster will be the proverbial chicken that lays the first egg. To start we will deploy a basic AKS cluster.

```bash
LOCATION=eastus2
CONTROLPLANE_GROUP=controlplane-rg
ASO_CLUSTER=asocluster-aks
SUBID=$(az account show -o tsv --query id)
TENANT=$(az account show -o tsv --query tenantId)
USERNAME=$(az account show -o tsv --query user.name)

az group create -n $CONTROLPLANE_GROUP -l $LOCATION
ASOID=$(az aks create -g $CONTROLPLANE_GROUP -n $ASO_CLUSTER --node-count 1 --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys -o tsv --query id)
```

ASO will need permission to create clusters and backing resources. To support this, we will create a user-assigned managed identity and delegate Contributor rights to a single resource group where we will deploy our workload clusters.

```bash
FLEET_GROUP=myfleet-rg
ASO_IDENTITY_NAME=aso-controlplane-identity
IDENTITY=$(az identity create -g $FLEET_GROUP -n $ASO_IDENTITY_NAME -o tsv --query clientId)
az role assignment create --role "Contributor" --assignee $IDENTITY --scope /subscriptions/$SUBID/resourceGroups/$FLEET_GROUP

# Establish trust between the ASO cluster service account for ASO and the UMI
ISSUER=$(az aks show -g $FLEET_HUB_GROUP -n $ASO_CLUSTER -o json | jq -r '.oidcIssuerProfile.issuerUrl')
az identity federated-credential create --name aso-federated-credential \
    --identity-name aso-aks-contributor -g $FLEET_HUB_GROUP \
    --issuer $ISSUER \
    --subject "system:serviceaccount:azureserviceoperator-system:azureserviceoperator-default" \
    --audiences "api://AzureADTokenExchange"
```

Now we just need to install flux and point it at the controlplane folder to bootstrap ASO onto the cluster.

```bash
az k8s-configuration flux create \ 
    -g $CONTROLPLANE_GROUP \ 
    -c $ASO_CLUSTER \ 
    -n cluster-config \ 
    --namespace cluster-config \ 
    -t managedClusters \ 
    --scope cluster \ 
    -u https://github.com/markjgardner/aks-multi-cluster-management \ 
    --branch flux-dev \ 
    --kustomization name=controlplane path=./flux/clusters/controlplane prune=true
```


### TODO: Remove this section
Once the cluster is finished provisioning, we need to install ASO.

```bash
# Install Azure Service Operator on the cluster
az aks get-credentials -g $FLEET_HUB_GROUP -n $ASO_CLUSTER --overwrite-existing

# ASO requires cert manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.14.1/cert-manager.yaml

# Helm install ASOv2
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
kubectl wait --namespace cert-manager --for=condition=available --timeout=600s deployment/cert-manager
helm upgrade --install aso2 aso2/azure-service-operator \
        --create-namespace \
        --namespace=azureserviceoperator-system \
        --set azureSubscriptionID=$SUBID \
        --set azureTenantID=$TENANT \
        --set azureClientID=$IDENTITY \
        --set useWorkloadIdentityAuth=true \
        --set crdPattern='resources.azure.com/*;containerservice.azure.com/*;kubernetesconfiguration.azure.com/*'
```

## Deploy Workload Clusters
