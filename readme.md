# Krateo FinOps Module Pricing Example
This repository contains a Composition Definition for Krateo that leverages the Krateo Composable FinOps components to integrate resources created through the Azure Operator and the pricing visualization flow. The composition creates a Virtual Machine on Azure through the Azure operator, visualizing it in the frontend through a card widget and custom form. The widget shows the pricing information retrieved through the FinOps module.

It achieves so through the following components:
 - [finops-composition-definition-parser](https://github.com/krateoplatformops/finops-composition-definition-parser): this component parses the annotations contained in this chart, with the name `focus-resource-name`;
 - [azure-pricing-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/azure-pricing-rest-dynamic-controller-plugin): plugin for the operator generator to gather data from the Azure pricing API;
 - [focus-data-presentation-azure](https://github.com/krateoplatformops/focus-data-presentation-azure): this composition definition integrates a custom resource for the generator operator that fetches data from the Azure pricing API and the custom resource for the FOCUS operator, which allows the fetched pricing information to be stored into the database.

Additionally, it can be configured to utilize the new FinOps page in the composition portal basic. This page can be configured to show a breakdown of the costs of the composition and the usage metrics.

## Summary

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Examples](#examples)
4. [Installation](#Installation)
5. [Cost and Usage Metrics](#cost-and-usage-metrics)

## Architecture
This composition definition uses the components from the following architecture:

<p align="center">
<img src="/_diagrams/architecture.png" width="800">
</p>

## Examples
The following figure shows an example of the pricing information retrieved from the Azure Pricing API, placed inside the side menu:
<p align="center">
<img src="/_diagrams/pricing_frontend.png" width="800">
</p>

<sub>Note: The `1 GB/Month` expenditure has been added through the API as an example of multiple values and will not appear when installing the composition.</sub>

## Installation
Install the cert-manager for the Azure operator:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.yaml
```

Install the Azure operator (Azure Service Operator v2):
```sh
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
helm repo update
helm upgrade --install aso2 aso2/azure-service-operator \
    --create-namespace \
    --namespace=azureserviceoperator-system \
    --set crdPattern='resources.azure.com/*;compute.azure.com/*;network.azure.com/*'
```

Create the secret with the credentials for the Azure operator
```sh
kubectl create ns azure-pricing-system
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
 name: aso-credential
 namespace: azure-pricing-system
stringData:
 AZURE_SUBSCRIPTION_ID: # insert your subscription id
 AZURE_TENANT_ID:       # insert your tenant id
 AZURE_CLIENT_ID:       # insert your client id
 AZURE_CLIENT_SECRET:   # insert your client secret
EOF
```

Install the [azure-pricing-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/azure-pricing-rest-dynamic-controller-plugin):
```sh
helm install azure-pricing-rest-dynamic-controller-plugin krateo/azure-pricing-rest-dynamic-controller-plugin -n azure-pricing-system
```

Install the [focus-data-presentation-azure](https://github.com/krateoplatformops/focus-data-presentation-azure) composition definition:
```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  annotations:
     "krateo.io/connector-verbose": "true"
  name: focus-data-presentation-azure
  namespace: azure-pricing-system
spec:
  chart:
    repo: focus-data-presentation-azure
    url: https://charts.krateo.io
    version: "0.1.0"
EOF
```

Install the composition for the [focus-data-presentation-azure](https://github.com/krateoplatformops/focus-data-presentation-azure), which creates the `DataPresentationAzure` resource, managed by the [rest-dynamic-controller](https://github.com/krateoplatformops/rest-dynamic-controller). Then, the [composition-dynamic-controller](https://github.com/krateoplatformops/composition-dynamic-controller) will install the FocusConfig for the [finops-operator-focus](https://github.com/krateoplatformops/finops-operator-focus), which will start the exporter/scraper flow.
```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-1-0
kind: FocusDataPresentationAzure
metadata:
  name: finops-example-azure-vm-pricing
  namespace: azure-pricing-system
spec:
  annotationKey: krateo-finops-focus-resource
  filter: serviceName eq 'Virtual Machines' and armSkuName eq 'Standard_B2s' and armRegionName eq 'westus3' and type eq 'Consumption'
  operatorFocusNamespace: krateo-system
  scraperConfig:
    tableName: pricing_table
    pollingInterval: "1h"
    scraperDatabaseConfigRef: 
      name: finops-database-handler
      namespace: krateo-system
EOF
```

Install the composition definition for this component:
```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  annotations:
     "krateo.io/connector-verbose": "true"
  name: finops-example-pricing-vm-azure
  namespace: azure-pricing-system
spec:
  chart:
    repo: finops-example-pricing-vm-azure
    url: https://charts.krateo.io
    version: "0.1.3"
EOF
```

Finally, install the resources for the frontend: [customform.yaml](https://github.com/krateoplatformops/finops-example-pricing-vm-azure/blob/main/customform.yaml).
```sh
kubectl apply -f customform.yaml
```

# Cost and Usage Metrics
This section will cover how to configure the Krateo Composable FinOps to collect cost and usage metrics from the virtual machine just created on Azure.

> [!NOTE]  
> The Azure Portal UI may change over time, so some steps or visuals might differ slightly from what is described here.

# Cost Metrics
To start, you will need a storage account to store the FOCUS exports. If you do not already have it, you need to configure it: 
1. go on your Azure dashboard and `Storage Accounts`, hit `Create` and select the `Blob Storage` mode. Then, follow the guided steps to complete the configuration. 
2. go to `Access Control (IAM)`, then `Role Assignments` and add a new role assignment. Search for `Azure Blob Data Owner` and hit forward. Select the application that you use for the authentication. Finally, assign it.

To configure the FOCUS exports on the storage account just created: 
1. go on your dashboard and search for `Cost Management`.
2. on the left pane, select `Exports`, then `Create`
3. select the FOCUS export and how often the export should run. 
4. select the storage account that you just created. 
5. to use the `Logic App` provided below, set the container to `focus`, the directory to `exports` and the format to `CSV`, without compression and with overwrite.

Now the export will run automatically to the storage account. However, it will place it inside a folder with the `uid`of the run, creating a dynamic path that cannot be known at runtime to configure the exporters and scrapers. Therefore, we will use a `Logic App` to copy the export from the `uid` folder to a static location.

Go back to the Azure dashboard and search for `Logic apps`. Then, create a new `Logic App` with the tier you most prefer (the consumption tier is pretty cheap if used for this task only). When created, hit `Edit`, then graphic editor will open. Select `Code view` and paste the following code. 

```json
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "contentVersion": "1.0.0.0",
        "triggers": {
            "Recurrence": {
                "type": "Recurrence",
                "recurrence": {
                    "frequency": "Day",
                    "interval": 1,
                    "schedule": {
                        "hours": [
                            1
                        ],
                        "minutes": [
                            0
                        ]
                    },
                    "timeZone": "UTC"
                }
            }
        },
        "actions": { }
    }
}
```

Select `Designer` to go back to the graphical view.

Then, create the following objects:
1. click on the plus below `Recurrence`, add an action, and on the right pane search `azure blob storage` and add `list blobs`, this will prompt to create a connection. Configure the connection to your storage account. Use the `Service principal authentication` and provide the client id and secret of the application to which you granted the `Azure Blob Data Owner` permissions (which is also the same you will use for the exporters' authentication). Set `Flat listing` to `Yes`.
2. add another action by searching for `data operations` and adding `filter array`. Click on parameters, then the lightning and finally `value`. Then, in `Filter query`, select the lightning and `Name`, then in the filter `starts with` and in the value `part_`.
3. add another `filter array`, in the data value choose the `fx` button, switch to `dynamic content` and select `Body`. Then, in the `Filter query` select `Body Name`, `ends with` and set the value to `.csv`.
4. add one more action of the type `data operations`, but this time choose `compose`. In the `inputs` click on the `fx` button and then write `sort()` in the box that opens, select dynamic content and click `Body`, then add `LastModified`. The output should look like this:
```
sort(body('<YOUR PREVIOUS ACTION NAME>', 'LastModified'))
```
5. finally, add one last action by serach for `azure blob storage` and selecting `copy blobs`. Set the storage account name to your storage account, then the destination blob to `/focus/exports/static-export.csv`. For the source url, click on the `fx` button. Write `last()` then select `Dynamic content` and choose `Outputs` from the previous `Compose` action. Then add `?['Path']` at the end to select the right field of the object. Your `fx` should look like this:
```
last(outputs('<YOUR COMPOSE ACTION NAME>'))?['Path']
```
Set `Overwrite` to `Yes`.

Now you can save and run the logic app. If you did everything right, you should see a successful run.

## Usage Metrics
No additional setup is required to obtain usage metrics from Azure.
The Helm chart already includes an `exporterscraperconfig.yaml` file among its templates, which contains the configuration for the exporters and scrapers related to usage metrics.

## Krateo Composable FinOps
To start the exporter/scraper for all costs and CPU usage of VMs, see the following [sample](/samples/exporterscrapersample.yaml). It also includes the configuration for the external-secrets operator, which handles retrieving bearer tokens from Azure. See the installation instructions here: [Getting started](https://external-secrets.io/latest/introduction/getting-started/).

Finally, enable the cluster-wide external secrets for the krateo-system namespace with:
```
kubectl label namespace krateo-system azure-secrets=enabled
```

# Output Sample
The FinOps tab will look like this when populated with metrics:

<p align="center">
<img src="/_diagrams/metrics_frontend.png" width="800">
</p>