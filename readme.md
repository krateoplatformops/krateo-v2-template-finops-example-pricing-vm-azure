# Krateo FinOps Module Pricing Example

This repository contains a Composition Definition for Krateo that integrates the Krateo Composable FinOps components and pricing visualization integration. It creates a Virtual Machine (VM) on Azure through the Azure operator. It also shows the template card with pricing information.

It achieves so through the following components, which can be explored to learn more:
 - [finops-composition-definiition-parser](https://github.com/krateoplatformops/finops-composition-definition-parser): this component parses the annotations contained in this chart, with the name `focus-resource-name`;
 - [azure-pricing-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/azure-pricing-rest-dynamic-controller-plugin): plugin for the operator generator to gather data from the Azure pricing API;
 - [focus-data-presentation-azure](https://github.com/krateoplatformops/focus-data-presentation-azure): this composition definition integrates a custom resource for the generator operator that fetches data from the Azure pricing API and the custom resource for the FOCUS operator, which allows the fetched pricing information to be stored into the database.

## Architecture
This composition definition uses the components from the following architecture:

![FinOps Data Presentation](_diagrams/architecture.png)

## installation
Install the cert-manager for the Azure operator:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.yaml
```

Install the Azure operator:
```
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
helm repo update
helm upgrade --install aso2 aso2/azure-service-operator \
    --create-namespace \
    --namespace=krateo-system \
    --set crdPattern='resources.azure.com/*;compute.azure.com/*;network.azure.com/*'
```

Create the secret with the credentials for the Azure operator
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
 name: aso-credential
 namespace: krateo-system
stringData:
 AZURE_SUBSCRIPTION_ID: 
 AZURE_TENANT_ID: 
 AZURE_CLIENT_ID: 
 AZURE_CLIENT_SECRET: 
EOF
```

Install Krateo's Operator generator
```
helm install krateo-oasgen-provider krateo/oasgen-provider -n krateo-system
```

Install the [azure-pricing-rest-dynamic-controller-plugin](https://github.com/krateoplatformops/azure-pricing-rest-dynamic-controller-plugin):
```
helm install azure-pricing-rest-dynamic-controller-plugin krateo/azure-pricing-rest-dynamic-controller-plugin -n krateo-system
```

Install the [focus-data-presentation-azure](https://github.com/krateoplatformops/focus-data-presentation-azure) composition definition:
```
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  annotations:
     "krateo.io/connector-verbose": "true"
  name: focus-data-presentation-azure
  namespace: krateo-system
spec:
  chart:
    repo: focus-data-presentation-azure
    url: https://charts.krateo.io
    version: "0.1.0"
EOF
```
