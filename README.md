# Connected Micro services with K8s and ServiceMesh



## Introduction

## Purpose

## What to Accomplish

- ### K8s High Level Architecture

- ### AKS - High Level Architecture

- ### AKS - Target Architecture

- ### Understand ServiceMesh - *What it is?*

  - #### Features

  - #### Benefits

    - ##### Challenges it Solves

- ### Let us Delve into it

  - #### Clone Workshop repo

  - #### Plan Workshop Tasks

  - #### Create Primary AKS Cluster

    - ##### Login to Azure
    
    - ##### Create Primary Resource Group

    - ##### Create Service Principal - Primary

    - ##### Create Network for AKS - Primary

      - ###### Create Virtual Network

      - ###### Create Subnet for AKS cluster

      - ###### Create Subnet for Ingress Controller

    - ##### Assign Roles to Service Principal

    - ##### Create Azure Container Registry (ACR) - Primary

    - ##### Create Primary AKS cluster

    - #### Cluster Configuration - Post Creation

      - ##### Create Namespaces

        - ###### primary

      - ##### Deploy RBAC

      - ##### Configure SSL (*Optional*)

        - ###### Get SSL Certificate

        - ###### Create K8s Secret

      - ##### Install Nginx Ingress Controller - Primary

      - ##### Deploy Ingress - Primary

        - ###### Configure Ingress with SSL

        - ###### Configure Routes

    - #### Deploy Microservices - Primary

      - ##### Deploy MongoDB Container

      - ##### Deploy Ratings Web app - V1, Staging

      - ##### Deploy Ratings Api app - V1, Staging

    - #### Test  Integration flow

    - #### Create Secondary AKS Cluster

      - ##### Create Secondary Resource Group

      - ##### Create Service Principal - Secondary 

      - ##### Create Network for AKS - Secondary

        - ###### Create Virtual Network

        - ###### Create Subnet for AKS cluster

        - ###### Create Subnet for Ingress Controller

      - ##### Assign Roles to Service Principal

      - ##### Create Azure Container Registry (ACR) - Secondary

      - ##### Create Secondary AKS cluster

      - #### Cluster Configuration - Post Creation

        - ##### Create Namespaces

          - ###### secondary

        - ##### Deploy RBAC

        - ##### Configure SSL (*Optional*)

          - ###### Get SSL Certificate

          - ###### Create K8s Secret

        - ##### Install Nginx Ingress Controller - Secondary

        - ##### Deploy Ingress - Secondary

          - ###### Configure Ingress with SSL

          - ###### Configure Routes

      - #### Deploy Microservices - Secondary

        - ##### Deploy MongoDB Container

        - ##### Deploy Ratings Web app - V2, Production

        - ##### Deploy Ratings Api app - V2, Production

      - #### Test  Integration flow

    - ### Service Mesh

      - #### Istio

        - ##### Download Istio

        - ##### Install Istio CLI - istioctl

        - ##### Configure Istio in Primary Cluster

          - ###### Install Istio onto Primary Cluster

          - ###### Inject Istio into Namespaces

          - ###### Install Kiali dashboard

          - ###### Observability

            - ###### View Basic Metrics

            - ###### View API Dependencies

            - ###### Scale Pods; view Metrics

          - ###### Traffic Splitting

          - ###### Fault Injection

          - ###### Circuit Breaking

          - ###### Distributed Tracing

          - ###### Service Mirroring

      - #### Linkerd

        - ##### Download Linkerd

        - ##### Install Linkerd CLI - linkerdctl

        - ##### Configure Linkerd in Primary Cluster

          - ###### Install Linkerd onto Primary Cluster

          - ###### Inject Linkerd into Namespaces

          - ###### Install Viz dashboard

          - ###### Install Jaeger for Distributed Tracing

          - ###### Observability

            - ###### View Basic Metrics

            - ###### View Detailed Metrics

              - ###### Grafana

            - ###### <u>*View API Dependencies*</u>??

            - ###### Scale Pods; view Metrics

          - ###### Traffic Splitting

          - ###### Fault Injection

          - ###### Circuit Breaking

          - ###### Distributed Tracing

          - ###### Service Mirroring

      - #### API Mesh

        - ##### Azure API Managemt Service

          - ###### What it is

          - ###### How it Integrates with AKS or K8s

            - ###### Outside the Cluster

            - ###### Inside the Cluster 

            - ###### End to End view

            - ###### Application Gateway - L7 Load Balancer

          - ##### Self Hosted APIM

            - ###### Features/Benefits

            - ###### Deploy APIM

            - ###### Enable Application Insights

            - ###### Add Authentication

            - ###### Create Self Hosted Gateway

            - ###### Deploy APIM Container on AKS cluster

          - ##### Configure APIs behind APIM

            - ###### Ratings Web

            - ###### Ratings Api

            - ###### Expose APIs

              - ###### Developer Portal

          - ##### Test Integration Flow

          - ##### View API metrics

            - ###### Application Insights

          - ##### Add API Policies

            - ###### Restrict API access

              - ###### RBAC

          - ##### How it is diff from Service Mesh approach

            - ###### What is Additional

            - ###### What is missing

        - ##### Sumamry of all Appriches

          - ###### Pros & Cons

        - ##### What Next

          - ###### Multi Cluster Connectivity

          - ###### Elastic Scaling

  - ## HOL

    - ### K8s High Level Architecture

      ![K8s-HL](./Assets/K8s-HL.png)

      

    - **AKS - High Level Architecture**

      ![K8s-HL](./Assets/aks-HL.png)

      

    - ### AKS - Target Architecture

      [TBD]

    - ### Understand ServiceMesh - *What it is?*

      ![istio-arch](./Assets/service-mesh.png)

      

      - #### Features

      - #### Benefits

        - ##### Challenges it Solves

    - ### Let us Delve into it

      - #### **Clone Workshop repo**

        ```bash
        git clone <url>
        ```

      - #### Plan Workshop Tasks

        - Define Local variables

          ```bash
          tenantId=""
          subscriptionId=""
          aksResourceGroup=""
          location=""
          clusterName=""
          version=""
          acrName=""
          acrId=
          aksVnetName=""
          aksVnetPrefix=""
          aksVnetId=
          aksSubnetName=""
          aksSubnetPrefix=""
          aksSubnetId=
          aksIngressSubnetName=""
          aksIngressSubnetPrefix=""
          aksIngressSubnetId=
          sysNodeSize="Standard_DS3_v2"
          sysNodeCount=3
          maxSysPods=30
          networkPlugin=azure
          networkPolicy=azure
          sysNodePoolName=akssyspool
          ingressNodePoolName=akssyspool
          vmSetType=VirtualMachineScaleSets
          addons=monitoring
          spAppId=""
          spPassword=""
          aksIngControllerName=""
          aksIngControllerNSName=""
          aksIngControllerFileName="internal-ingress"
          backendIpAddress=
          ```

        - Login to Azure

          ```bash
          #Login to Azure
          az login --tenant $tenantId
          
          #Check Selected Subscription
          az account show
          
          #Set appropriate Subscription, if needed
          #az account set -s $subscriptionId
          ```

        - Create Primary Resource Group

          ```bash
          #Create Resource Group for AKS workloads
          az group create -n $aksResourceGroup -l $location
          ```

        - Service Principal

          ```bash
          #Create Service Principal
          az ad sp create-for-rbac --skip-assignment -n https://aks-oprimary-sp
          {
            "appId": "",
            "displayName": "https://arc-aks-sp",
            "name": "",
            "password": "",
            "tenant": ""
          }
          
          #Create Service Principal
          az ad sp create-for-rbac --skip-assignment -n https://aks-train-sp
          {
            "appId": "",
            "displayName": "https://arc-aks-sp",
            "name": "",
            "password": "",
            "tenant": ""
          }
          
          #Set Service Principal variables
          spAppId=""
          spPassword=""
          ```

      - #### Create Network for AKS - Primary

        - ##### **Create Virtual Network**

          ```bash
          #Deploy Spoke Virtual Network
          az network vnet create -n $aksVnetName -g $aksResourceGroup --address-prefixes $aksVnetPrefix
          aksVnetId=$(az network vnet show -n $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
          echo $aksVnetId
          ```

          

        - ##### **Create Subnet for AKS cluster**

          ```bash
          #Deploy AKS Subnet inside Virtual Network
          az network vnet subnet create -n $aksSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --address-prefixes $aksSubnetPrefix
          aksSubnetId=$(az network vnet subnet show -n $aksSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
          echo $aksSubnetId
          
          ```

        - ##### **Create Subnet for Ingress Controller**

          ```bash
          #Deploy Ingress Subnet inside Virtual Network
          az network vnet subnet create -n $aksIngressSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --address-prefixes $aksIngressSubnetPrefix
          aksIngressSubnetId=$(az network vnet subnet show -n $aksIngressSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
          echo $aksIngressSubnetId
          ```

      - #### Assign Roles to Service Principal

        ```bash
        #Assign Role to Spoke Virtual Network
        az role assignment create --assignee $spAppId --role "Network Contributor" --scope $aksVnetId
        ```

      - #### Create Azure Container Registry (ACR) - Primary

        ```bash
        #Deploy ACR
        az acr create -n $acrName -g $aksResourceGroup --sku STANDARD --admin-enabled false
        acrId=$(az acr show -n $acrName -g $aksResourceGroup --query="id" -o tsv)
        echo $acrId
        
        #Assign Role to Service Principal for the ACR
        az role assignment create --assignee $spAppId --role "AcrPull" --scope $acrId
        ```

        

      - #### Create Primary AKS Cluster

        ```bash
        #Create Public AKS cluster
        az aks create --name $clusterName \
        --resource-group $aksResourceGroup \
        --kubernetes-version $version --location $location \
        --vnet-subnet-id "$aksSubnetId" --enable-addons $addons \
        --node-vm-size $sysNodeSize \
        --node-count $sysNodeCount --max-pods $maxSysPods \
        --service-principal $spAppId \
        --client-secret $spPassword \
        --network-plugin $networkPlugin --network-policy $networkPolicy \
        --nodepool-name $sysNodePoolName --vm-set-type $vmSetType \
        --generate-ssh-keys \
        --enable-aad \
        --aad-admin-group-object-ids $aadAdminGroupID \
        --aad-tenant-id $aadTenantID \
        --attach-acr $acrName
        ```
        
      - #### Authentication

        ```bash
        #Connect to AKS cluster and check status
        az aks get-credentials -g $aksResourceGroup --name $clusterName --admin --overwrite
        kubectl get ns
        
        #Connect to AKS cluster as Admin
        az aks get-credentials -g $resourceGroup -n $clusterName --admin
        ```

        

      - #### Cluster Configuration - Post Creation

        - ##### Create Namespaces

          - ###### primary

            ```bash
            kubectl get ns primary
            ```

          - ##### **Configure SSL (***Optional*)

            - ###### Get SSL Certificate

            - ###### Create K8s Secret

          - ##### Deploy Microservices - Primary

            - Deploy MongoDB Container

              ```bash
              # Add Helm repo for MongoDB
              helm repo add bitnami https://charts.bitnami.com/bitnami
              helm search repo bitnami
              
              # Install MongoDB with Helm chart
              helm install ratingsdb bitnami/mongodb -n db \
              --set auth.username=ratingsuser,auth.password=ratingspwd,auth.database=ratingsdb \
              --set controller.nodeSelector.agentpool=akssyspool \
              --set controller.defaultBackend.nodeSelector.agentpool=akssyspool
              ```

            - Create necessary Secrets for MongoDB connection

              ```bash
              kubectl create secret generic aks-workshop-mongo-secret -n aks-workshop-dev \
                  --from-literal=MONGOCONNECTION="mongodb://ratingsuser:ratingspwd@ratingsdb-mongodb.db:27017/ratingsdb"
              ```

              

            - ###### Deploy Ratings Web app - V1, Staging

              - Build RatingsWeb Image and Push to ACR

                ```bash
                # Build and Push ACR image for RatingsWeb
                az acr build -t $acrName.azurecr.io/ratings-web:v1.0.0 -r $acrName .
                ```

              - Modify ratingsweb-chart/values-dev.yaml

                ```yaml
                [TBD]
                ```

              - Deploy RatingsWeb 

                ```bash
                # Install RatingsWeb with Helm chart
                helm install ratingsweb-chart -n aks-workshop-dev ./ratingsweb-chart/ -f ./ratingsweb-chart/AKSWorkshop/values-dev.yaml
                ```

            - ###### Deploy Ratings Api app - V1, Staging

              - Build RatingsWeb Image and Push to ACR

                ```bash
                # Build and Push ACR image for RatingsWeb
                az acr build -t $acrName.azurecr.io/ratings-api:v1.0.0 -r $acrName .
                ```

              - Modify ratingsapi-chart/values-dev.yaml

                ```
                [TBD]
                ```

              - Deploy RatingsApi 

                ```bash
                helm install ratingsapi-chart -n aks-workshop-dev ./ratingsapi-chart/ -f ./ratingsapi-chart/AKSWorkshop/values-dev.yaml
                ```

            - ##### Test  Integration flow

              - [TBD]

                

      - #### Create Secondary AKS Cluster

        ```bash
        # Follow similar steps as Primary
        ```

        - ##### Deploy Microservices - Secondary

          - Deploy MongoDB Container

            ```bash
            # Add Helm repo for MongoDB
            helm repo add bitnami https://charts.bitnami.com/bitnami
            helm search repo bitnami
            
            # Install MongoDB with Helm chart
            helm install ratingsdb bitnami/mongodb -n db \
            --set auth.username=ratingsuser,auth.password=ratingspwd,auth.database=ratingsdb \
            --set controller.nodeSelector.agentpool=akssyspool \
            --set controller.defaultBackend.nodeSelector.agentpool=akssyspool
            ```

          - Create necessary Secrets for MongoDB connection

            ```bash
            kubectl create secret generic aks-workshop-mongo-secret -n aks-workshop-dev \
                --from-literal=MONGOCONNECTION="mongodb://ratingsuser:ratingspwd@ratingsdb-mongodb.db:27017/ratingsdb"
            ```

            

          - ###### Deploy Ratings Web app - V2, P

            - Build RatingsWeb Image and Push to ACR

              ```bash
              # Build and Push ACR image for RatingsWeb
              az acr build -t $acrName.azurecr.io/ratings-web:v1.0.0 -r $acrName .
              ```

            - Modify ratingsweb-chart/values-dev.yaml

              ```yaml
              [TBD]
              ```

            - Deploy RatingsWeb 

              ```bash
              # Install RatingsWeb with Helm chart
              helm install ratingsweb-chart -n aks-workshop-dev ./ratingsweb-chart/ -f ./ratingsweb-chart/AKSWorkshop/values-dev.yaml
              ```

          - ###### Deploy Ratings Api app - V1, Staging

            - Build RatingsWeb Image and Push to ACR

              ```bash
              # Build and Push ACR image for RatingsWeb
              az acr build -t $acrName.azurecr.io/ratings-api:v1.0.0 -r $acrName .
              ```

            - Modify ratingsapi-chart/values-dev.yaml

              ```
              [TBD]
              ```

            - Deploy RatingsApi 

              ```bash
              helm install ratingsapi-chart -n aks-workshop-dev ./ratingsapi-chart/ -f ./ratingsapi-chart/AKSWorkshop/values-dev.yaml
              ```

          - ##### Test  Integration flow

            - [TBD]

    - ### Service Mesh

      - #### Istio

        ![istio-arch](./Assets/istio-arch.png)

        - ##### Download Istio

          ```bash
          curl -L https://istio.io/downloadIstio | sh -
          
          #Set IstioPath env variable to Istio folder path
          istioPath="/Users/monojitdattams/Development/Projects/Workshops/ServiceMesh_Workshop/Istio/istio-1.11.3"
          ```

        - ##### Configure Kubctl context on Primary

          ```bash
          export CTX_CLUSTER1=primary
          
          #Set Context to primary
          az aks get-credentials -g $resourceGroup -n $clusterName --context $CTX_CLUSTER1
          
          #Check cluster health
          kubectl get no
          ```

        - ##### Install Istio CLI on Primary - istioctl

          ```bash
          #Select Default Istio Profile settings
          istioctl install --set profile=default -y --context=$CTX_CLUSTER1
          ```

        - ##### Configure Istio in Primary Cluster

          - ###### Inject Istio into Namespaces

            ```bash
            kubectl label namespace primary istio-injection=enabled
            ```

          - ##### Install Kiali dashboard

            ```bash
            kubectl apply -f $istioPath/samples/addons
            
            #Check Kiali deployment
            kubectl rollout status deployment/kiali -n istio-system
            istioctl dashboard kiali&
            ```

          - ##### Observability

            - ###### View Basic Metrics

            - ###### View API Dependencies

            - ###### Scale Pods; view Metrics

              

        - ##### Configure Kubctl context - Secondary

          ```bash
          export CTX_CLUSTER2=secondary
          
          #Set Context to secondary
          az aks get-credentials -g $resourceGroup -n $clusterName --context $CTX_CLUSTER2
          
          #Check cluster health
          kubectl get no
          ```

        - ##### Install Istio CLI on Secondary - istioctl

          ```bash
          #Select Default Istio Profile settings
          istioctl install --set profile=default -y --context=$CTX_CLUSTER2
          ```

        - ##### Configure Istio in Secondary Cluster

          - ###### Inject Istio into Namespaces

            ```bash
            kubectl label namespace secondary istio-injection=enabled
            ```

          - ##### Install Kiali dashboard

            ```bash
            kubectl apply -f $istioPath/samples/addons
            
            #Check Kiali deployment
            kubectl rollout status deployment/kiali -n istio-system
            istioctl dashboard kiali&
            ```

          - ##### Observability

            - ###### View Basic Metrics

            - ###### View API Dependencies

            - ###### Scale Pods; view Metrics

          

        - ###### Traffic Splitting

          ![istio-trafficplit](./Assets/istio-trafficplit.png)

          ```
          [TBD]
          ```

          

        - ###### Fault Injection

          ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)

          ```
          [TBD]
          ```

          

        - ###### Circuit Breaking

        - ###### Distributed Tracing

        - ###### Service Mirroring

          ![istio-trafficplit](./Assets/istio-mirroring-2.png)

          

          ![istio-trafficplit](./Assets/istio-mirroring-3.png)

          ![istio-trafficplit](./Assets/istio-mirroring.png)

          ```
          [TBD]
          ```

          

      - #### Linkerd

        ![istio-arch](./Assets/linkerd-arch.png)

        - ##### Download Linkerd

          ```bash
          #Export Linkerd version into an eb variable
          export LINKERD2_VERSION=stable-2.10.0
          
          #Install above version
          curl -sL https://run.linkerd.io/install | sh
          
          #Check installation status
          linkerd check --pre
          linkerd version
          ```

        - ##### Install Linkerd CLI - linkerdctl

          ```bash
          linkerd install  | kubectl apply -f -
          
          #Check installation status
          linkerd check
          ```

          

        - ##### Configure Linkerd in Primary Cluster

          - ###### Inject Linkerd into Primary Namespace

            ```bash
            kubectl get ns/primary -o yaml | linkerd inject - | kubectl apply -f -
            ```

          - ##### Install Viz dashboard

            ```bash
            linkerd viz install | k apply -f -
            linkerd check
            linkerd viz dashboard&
            ```

          - ##### Install Jaeger for Distributed Tracing

            ```bash
            linkerd jaeger install | k apply -f -
            linkerd jaeger check
            linkerd jaeger dashboard&
            ```

          - ##### Observability

            - ###### View Basic Metrics

            - ###### View API Dependencies

            - ###### Scale Pods; view Metrics

          - ###### Traffic Splitting

            ![istio-trafficplit](./Assets/istio-trafficplit.png)

            ```
            [TBD]
            ```

            

          - ###### Fault Injection

            ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)

            ```
            [TBD]
            ```

            

          - ###### Circuit Breaking

            ```
            [TBD]
            ```

            

          - ###### Distributed Tracing

            ```
            [TBD]
            ```

            

          - ###### Service Mirroring

            ![linkerd-mirroring](./Assets/linkerd-mirroring.png)

  - ## References
