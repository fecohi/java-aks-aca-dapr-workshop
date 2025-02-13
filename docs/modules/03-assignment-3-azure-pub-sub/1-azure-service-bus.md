---
title: Using Dapr for pub/sub with Azure Service Bus
parent: Assignment 3 - Using Dapr for pub/sub with Azure Services
has_children: false
nav_order: 1
layout: default
has_toc: true
---

# Using Dapr for pub/sub with Azure Service Bus

{: .no_toc }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
- TOC
{:toc}
</details>

Stop Simulation, TrafficControlService and FineCollectionService, and VehicleRegistrationService by pressing Crtl-C in the respective terminal windows.

## Step 1: Create Azure Service Bus 

In this assignment, you will use Azure Service Bus as the message broker with the Dapr pub/sub building block. You're going to create an Azure Service Bus namespace and a topic in it. To be able to do this, you need to have an Azure subscription. If you don't have one, you can create a free account at [https://azure.microsoft.com/free/](https://azure.microsoft.com/free/).

1. Login to Azure:

    ```bash
    az login
    ```

1. Create a resource group:

    ```bash
    az group create --name rg-dapr-workshop-java --location eastus
    ```

    A [resource group](https://learn.microsoft.com/azure/azure-resource-manager/management/manage-resource-groups-portal) is a container that holds related resources for an Azure solution. The resource group can include all the resources for the solution, or only those resources that you want to manage as a group. In our workshop, all the databases, all the microservices, etc. will be grouped into a single resource group.

1. [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/) Namespace is a logical container for topics, queues, and subscriptions. This namespace needs to be globally unique. Use the following command to generate a unique name:

    - Linux/Unix shell:
       
        ```bash
        UNIQUE_IDENTIFIER=$(LC_ALL=C tr -dc a-z0-9 </dev/urandom | head -c 5)
        SERVICE_BUS="sb-dapr-workshop-java-$UNIQUE_IDENTIFIER"
        echo $SERVICE_BUS
        ```

    - Powershell:
    
        ```powershell
        $ACCEPTED_CHAR = [Char[]]'abcdefghijklmnopqrstuvwxyz0123456789'
        $UNIQUE_IDENTIFIER = (Get-Random -Count 5 -InputObject $ACCEPTED_CHAR) -join ''
        $SERVICE_BUS = "sb-dapr-workshop-java-$UNIQUE_IDENTIFIER"
        $SERVICE_BUS
        ```

1. Create a Service Bus messaging namespace:

    ```bash
    az servicebus namespace create --resource-group rg-dapr-workshop-java --name $SERVICE_BUS --location eastus
    ```

1. Create a Service Bus topic:

    ```bash
    az servicebus topic create --resource-group rg-dapr-workshop-java --namespace-name $SERVICE_BUS --name test
    ```

1. Create authorization rules for the Service Bus topic:

    ```bash
    az servicebus topic authorization-rule create --resource-group rg-dapr-workshop-java --namespace-name $SERVICE_BUS --topic-name test --name DaprWorkshopJavaAuthRule --rights Manage Send Listen
    ```

1. Get the connection string for the Service Bus topic and copy it to the clipboard:

    ```bash
    az servicebus topic authorization-rule keys list --resource-group rg-dapr-workshop-java --namespace-name $SERVICE_BUS --topic-name test --name DaprWorkshopJavaAuthRule  --query primaryConnectionString --output tsv
    ```

## Step 2: Configure the pub/sub component

1. Open the file `dapr/azure-servicebus-pubsub.yaml` in your code editor.

    ```yaml
    apiVersion: dapr.io/v1alpha1
    kind: Component
    metadata:
      name: pubsub
    spec:
      type: pubsub.azure.servicebus
      version: v1
      metadata:
      - name: connectionString # Required when not using Azure Authentication.
        value: "Endpoint=sb://{ServiceBusNamespace}.servicebus.windows.net/;SharedAccessKeyName={PolicyName};SharedAccessKey={Key};EntityPath={ServiceBus}"
    scopes:
      - trafficcontrolservice
      - finecollectionservice
    ```

    As you can see, you specify a different type of pub/sub component (`pubsub.azure.servicebus`) and you specify in the `metadata` section how to connect to Azure Service Bus created in step 1. For this workshop, you are going to use the connection string you copied in the previous step. You can also configure the component to use Azure Active Directory authentication. For more information, see [Azure Service Bus pub/sub component](https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-azure-servicebus-topics/).

    In the `scopes` section, you specify that only the TrafficControlService and FineCollectionService should use the pub/sub building block.

1. **Copy or Move** this file `dapr/azure-servicebus-pubsub.yaml` to `dapr/components` folder.

1. **Replace** the `connectionString` value with the value you copied from the clipboard.

1. **Move** the files `dapr/components/kafka-pubsub.yaml` and `dap/components/rabbit-pubsub.yaml`  back to `dapr/` folder if they are present in the component folder.

## Step 3: Test the application

You're going to start all the services now. 

1. Make sure no services from previous tests are running (close the command-shell windows).

1. Open the terminal window and make sure the current folder is `VehicleRegistrationService`.

1. Enter the following command to run the VehicleRegistrationService with a Dapr sidecar:

   ```bash
   mvn spring-boot:run
   ```

1. Open a terminal window and change the current folder to `FineCollectionService`.

1. Enter the following command to run the FineCollectionService with a Dapr sidecar:

   ```bash
   dapr run --app-id finecollectionservice --app-port 6001 --dapr-http-port 3601 --dapr-grpc-port 60001 --components-path ../dapr/components mvn spring-boot:run
   ```

1. Open a terminal window and change the current folder to `TrafficControlService`.

1. Enter the following command to run the TrafficControlService with a Dapr sidecar:

   ```bash
   dapr run --app-id trafficcontrolservice --app-port 6000 --dapr-http-port 3600 --dapr-grpc-port 60000 --components-path ../dapr/components mvn spring-boot:run
   ```

1. Open a terminal window and change the current folder to `Simulation`.

1. Start the simulation:

   ```bash
   mvn spring-boot:run
   ```

You should see the same logs as before. Obviously, the behavior of the application is exactly the same as before. But now, instead of messages being published and subscribed via kafka topic, are being processed through Azure Service Bus.

<span class="fs-3">
[< Assignment 2 - Run with Dapr]({{ site.baseurl }}{% link modules/02-assignment-2-dapr-pub-sub/index.md %}){: .btn .mt-7 }
</span>
<span class="fs-3">
[Assignment 4 - Observability >]({{ site.baseurl }}{% link modules/04-assignment-4-observability-zipkin/index.md %}){: .btn .float-right .mt-7 }
</span>
