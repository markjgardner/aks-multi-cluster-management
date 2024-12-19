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
ASO_IDENTITY_NAME=aso-controlplane-identity
IDENTITY=$(az identity create -g $CONTROLPLANE_GROUP -n $ASO_IDENTITY_NAME -o tsv --query clientId)
az role assignment create --role "Contributor" --assignee $IDENTITY --scope /subscriptions/$SUBID/resourceGroups/$CONTROLPLANE_GROUP

# Establish trust between the ASO cluster service account for ASO and the UMI
ISSUER=$(az aks show -g $CONTROLPLANE_GROUP -n $ASO_CLUSTER -o json | jq -r '.oidcIssuerProfile.issuerUrl')
az identity federated-credential create --name aso-federated-credential \
    --identity-name $ASO_IDENTITY_NAME -g $CONTROLPLANE_GROUP \
    --issuer $ISSUER \
    --subject "system:serviceaccount:azureserviceoperator-system:azureserviceoperator-default" \
    --audiences "api://AzureADTokenExchange"
```

In order for ASO to use workload identity for resource management, it needs to know the subscription, tenant and clientID to use. We will store those in the cluster as a secret which we can reference in the ASO deployment.
```bash
az aks get-credenitals -g $CONTROLPLANE_GROUP -n $ASO_CLUSTER --overwrite-existing
kubectl create secret generic aso-identity -n cluster-config --from-literal=values.yaml="
azureSubscriptionID: $SUBID
azureTenantID: $TENANT
azureClientID: $IDENTITY
"
```

Now we just need to install flux and point it at the controlplane repository to bootstrap ASO onto the cluster.

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
    --kustomization name=controlplane path=./flux/controlplane prune=true
```

## Deploy Workload Clusters

First we will create a resource group to hold the workload clusters and delegate access to the ASO identity allowing it to create new clusters.

```bash
FLEET_GROUP=myfleet-rg
az group create -n $FLEET_GROUP -l $LOCATION
az role assignment create --role "Contributor" --assignee $IDENTITY --scope /subscriptions/$SUBID/resourceGroups/$FLEET_GROUP
```

Workload cluster definitions can be placed in the [/flux/clusters] folder. To deploy new clusters whenever a resource definition is pushed to this folder we need to add another kustomization to the controlplane.

```bash
az k8s-configuration flux kustomization create \
    -g $CONTROLPLANE_GROUP \
    -c $ASO_CLUSTER \
    -n cluster-config \
    -t managedClusters \
    -k clusters \
    --path ./flux/clusters \
    --prune true \
    --interval 1m
```