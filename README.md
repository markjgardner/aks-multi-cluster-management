# Multi-Cluster AKS Management using Flux and ASO
This repo provides a sample implementation using [Flux](hhttps://fluxcd.io/) and [Azure Service Operator](https://azure.github.io/azure-service-operator/) to manage a dynamic fleet of Azure Kubernetes Clusters.

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
    --branch main \
    --kustomization name=controlplane path=./flux/controlplane prune=true
```

## Deploy Workload Clusters

First we will create a resource group to hold the workload clusters and delegate access to the ASO identity allowing it to create new clusters.

```bash
FLEET_GROUP=myfleet-rg
az group create -n $FLEET_GROUP -l $LOCATION
az role assignment create --role "Contributor" --assignee $IDENTITY --scope /subscriptions/$SUBID/resourceGroups/$FLEET_GROUP
```

Workload cluster definitions can be placed in the [/flux/clusters](/flux/clusters) folder. To deploy new clusters whenever a resource definition is pushed to this folder we need to add another kustomization to the controlplane.

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

## Cluster Definitions as Helm Charts

If we are thinking about our clusters as just abstract compute platforms for running our applications, the next logical leap is to just create templates for defining those environments. That way we can stamp out new environments and scale up and down easily by just reusing the template to create more or fewer clusters. [Helm](https://helm.sh) is one of the most robust templating tools for kubernetes resources and is well supported by Flux. So, let's see what a cluster helmchart looks like.

> [!NOTE]
> The full source for this chart can be found under [/charts/podinfocluster](/charts/podinfocluster).
```yaml
apiVersion: containerservice.azure.com/v1api20240901
kind: ManagedCluster
metadata:
  name: {{ $.Release.Name }}-{{ $i }}-aks
spec:
  location: {{ $.Values.location }}
  owner:
    name: myfleet-rg
  agentPoolProfiles:
    - name: system
      enableAutoScaling: true
      minCount: 1
      maxCount: 3
      vmSize: standard_d2as_v6
      osType: Linux
      osSKU: AzureLinux
      mode: System
  identity:
    type: SystemAssigned
  dnsPrefix: {{ $.Release.Name }}-{{ $i }}-aks
```

These workload clusters can be easily configured with their own flux configurations that load the application(s) onto the clusters.
```yaml
apiVersion: kubernetesconfiguration.azure.com/v1api20230501
kind: FluxConfiguration
metadata:
  name: flux-config
spec:
  gitRepository:
    repositoryRef:
      branch: main
    url: {{ $.Values.flux.repository }}
  kustomizations:
    apps: 
      dependsOn: 
        - infra
      force: false
      path: {{ $.Values.flux.kustomizationPath }}
      prune: true
      syncIntervalInSeconds: 600
      timeoutInSeconds: 600
      wait: true
    infra:
      force: false
      path: ./infrastructure
      prune: true
      syncIntervalInSeconds: 600
      timeoutInSeconds: 600
      wait: true
  namespace: cluster-config
  owner:
    group: containerservice.azure.com
    kind: ManagedCluster
    name: {{ $.Release.Name }}-{{ $i }}-aks
  sourceKind: GitRepository
```

To deploy instances of workload clusters, we just need to create a HelmRelease within our controlplane repo. Notice that you can create additional environments by defining new releases of the same chart. Scaling a particular instance (adding/removing clusters) can be accomplished by providing a target cluster count and letting helm render the appropriate number of resources on the controlplane cluster.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfoprod
  namespace: cluster-config
spec:
  releaseName: podinfo-prod
  targetNamespace: clusters
  chartRef:
    kind: OCIRepository
    name: aso-cluster-charts
  interval: 5m
  values:
    clusterCount: 3
    location: eastus2
    flux:
      kustomizationPath: ./apps/production
```