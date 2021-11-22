



# Connected Micro services with K8s and ServiceMesh



## Table Of Contents

- **K8s High Level Architecture**
- **AKS - High Level Architecture**
- **Understand ServiceMesh -** **What it is?**
  - **Features**
  - **Benefits**
- **Service Mesh - Where is it Deployed?**
- **Purpose**
- **What to Accomplish**
- **HOL**
  - **Pre-Requisites**
  - **Let us Delve into it**
  - **Clone Workshop repo**
  - **Plan Workshop Tasks**
    - **Define Local variables**
    - **Login to Azure**
    - **Create Resource Group**
    - **Service Principal**
    - **Create Network for AKS**
    - **Create Azure Container Registry (ACR) - Primary**
    - **Create Primary AKS Cluster**
    - **Create Secondary AKS Cluster**
  - **Service Mesh**
    - **Istio**
    - **Set CLI Variables for Istio**
    - **Configure AKS Cluster - Primary**
    - **Download Istio**
    - **Check Cluster Health**
    - **Create Namespaces**
    - **Install Istio CLI**
    - **Configure Istio in Primary Cluster**
      - **Inject Istio into Namespaces**
      - **Install Addons**
      - **Deploy BookInfo App**
      - **Expose Microservices thru an Ingress Gateway**
      - **Expose Kiali dashboard service thru a Gateway**
      - **Observability**
      - **Deploy more apps (Optional)**
      - **Traffic Splitting**
      - **Gateway**
      - **Virtual Service**
      - **Destination Rule**
      - **Fault Injection**
        - **Introduce Fault**
        - **Introduce Fix**
      - **Circuit Breaker**
      - **Distributed Tracing**
      - **Service Mirroring**
        - **Create and Configure Secondary Cluster**
        - **Configure Primary to Mirror services**
      - **CleanUP**



## Introduction

- ### K8s High Level Architecture

  ![K8s-HL](./Assets/K8s-HL.png)

  

- ### AKS - High Level Architecture

  ![aks-hl](./Assets/aks-hl.png)

  

  

- ### Understand ServiceMesh - *What it is?*

  ![istio-arch](./Assets/service-mesh.png)

  

  - #### Features

    - Observability
    - Ingress Routing
    - Traffic Management
      - Ingress Routing
      - Traffic Splitting
      - Distruibuted Tracing
      - Service Mirroring
      - Circuit Breaking

  - #### Benefits

    - Blue/Green Deployment
    - Fault Injection
    - Multi Cluster Connectivity
    - A/B Testing
  
- ### Service Mesh - Where is it Deployed?

  ![aks-hl](./Assets/aks-ref-arch.png)

  - Some Service Mesh can have its own Ingress Controller or Gateway - viz. *Istio*
    - Istio works in both ways
    - This workshop rather use Istio's own Gateway rather than requiring an existing one
  - Some Service Mesh might be dependent on existing Ingress Gateway - viz. *Linkerd*
    - Thsi workshop would use Linkerd with Nginx Ingress controller

## Purpose

- A view of Service Mesh technology
- Insights into the features/benefits it provides 
- No preference on particular tools/resource - open discussion
  - Istio/Linkerd is used just as a medium of comunication
- Provide a hands-on experience

## What to Accomplish

### ![workflow](./Assets/workflow.png)

- ## HOL

  - ### Pre-Requisites

    - Azure CLI
    - VSCode *(preferred)*
    - Github client
    - Any form of K8s Cluster. AKS is used here as an example

  - ### Let us Delve into it

    - #### **Clone Workshop repo**

      ```bash
      git clone https://github.com/monojit18/ServiceMeshWorkshop
      ```

    - #### Plan Workshop Tasks

      - ##### Define Local variables

        ```bash
        tenantId=""
        subscriptionId=""
        location="eastus"
        clusterName="primary-mesh-cluster"
        version="1.20.9"
        aksResourceGroup="primary-workshop-rg"
        acrName="srvmeshacr"
        spDisplayName="http://service-mesh-aks-cluster-sp"
        aksVnetName="primary-cluster-vnet"
        aksVnetPrefix=""
        aksVnetId=
        aksSubnetName="primary-cluster-subnet"
        aksSubnetPrefix=""
        aksSubnetId=
        aksServicePrefix=""
        dnsServiceIP=""
        sysNodeSize="Standard_DS2_v2"
        sysNodeCount=3
        maxSysPods=30
        networkPlugin=azure
        networkPolicy=azure
        sysNodePoolName=akssyspool
        vmSetType=VirtualMachineScaleSets
        addons=monitoring
        baseFolderPath=""
        ```
        
      - ##### Login to Azure
      
        ```bash
        #Login to Azure
        az login --tenant $tenantId
        
        #Check Selected Subscription
        az account show
        
        #Set appropriate Subscription, if needed
        #az account set -s $subscriptionId
        ```
      
      - ##### Create Resource Group
      
        ```bash
        #Create Resource Group for AKS workloads
        az group create -n $aksResourceGroup -l $location
        ```

      - ##### Service Principal
      
        ```bash
        #Create Service Principal
        az ad sp create-for-rbac --skip-assignment -n $spDisplayName
        {
          "appId": "",
          "displayName": "",
          "name": "",
          "password": "",
          "tenant": ""
        }
        
        #Set Service Principal variables
        spAppId=""
        spPassword=""
        ```
      
    - #### Create Network for AKS

      - ##### Create Virtual Network

        ```bash
        #Deploy Virtual Network
        az network vnet create -n $aksVnetName -g $aksResourceGroup --address-prefixes $aksVnetPrefix
        aksVnetId=$(az network vnet show -n $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
        echo $aksVnetId
        ```

      - ##### Create Subnet for AKS cluster

        ```bash
      #Deploy AKS Subnet inside the Virtual Network
        az network vnet subnet create -n $aksSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --address-prefixes $aksSubnetPrefix
        aksSubnetId=$(az network vnet subnet show -n $aksSubnetName --vnet-name $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
        echo $aksSubnetId
        
        #Assign Role to the Virtual Network
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
      --service-cidr $aksServicePrefix --dns-service-ip $dnsServiceIP \
      --node-vm-size $sysNodeSize \
      --node-count $sysNodeCount --max-pods $maxSysPods \
      --service-principal $spAppId \
      --client-secret $spPassword \
      --network-plugin $networkPlugin --network-policy $networkPolicy \
      --nodepool-name $sysNodePoolName --vm-set-type $vmSetType \
      --generate-ssh-keys \
      --disable-rbac \
      --attach-acr $acrName
      ```
      
      
      
    - #### Create Secondary AKS Cluster
    
      ```bash
      #Follow similar steps as Primary
      
      #Create AKS Cluster from CLI or Portal
      ```
    
  - ### Service Mesh

    - #### Istio

      ![istio-arch](./Assets/istio-arch.png)

      

      

      ##### Set CLI Variables for Istio

      ```bash
      primaryResourceGroup=$aksResourceGroup
      primaryClusterName=$clusterName
      secondaryResourceGroup="secondary-workshop-rg"
      secondaryClusterName="secondary-mesh-cluster"
      primaryAcrName=$acrName
      helmPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/AKS/Helm"
      istioPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/Istio"
      ```

      ##### Configure AKS Cluster - Primary

      ```bash
      #Set Env Variable for Primary Cluster
      #This helps to switch context easily between multiple clusters
      export CTX_CLUSTER1=primary
      
      #Connect to Public AKS Cluster with Primary Context
      az aks get-credentials -g $aksResourceGroup -n $clusterName --context $CTX_CLUSTER1
      
      #Additional helpful commands
      #Switch context between multiple clusters
      kubectl config use-context <context>
      ```

      ##### Download Istio

      ```bash
      #Download Istio binary
      curl -L https://istio.io/downloadIstio | sh -
      
      #Download specific version of Istio viz. 1.11.3
      curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.11.3 TARGET_ARCH=x86_64 sh -
      
      #The istioctl client binary in the bin/ directory
      #Add the istioctl client to your path (Linux or macOS):
      export PATH=$PWD/bin:$PATH
      ```

      ##### Check Cluster Health

      ```bash
      kubectl get no --context=$CTX_CLUSTER1
      kubectl get ns --context=$CTX_CLUSTER1
      ```

      ##### Create Namespaces

      ```bash
      #Create namespaces for Istio
      kubectl create namespace istio-system --context $CTX_CLUSTER1
      kubectl create namespace primary --context $CTX_CLUSTER1
      ```

      ##### Install Istio CLI

      ```bash
      #Install Istio CLI
      #Select Default Istio Profile settings
      #Ingress Gateway with Public IP Address
      istioctl install --context=$CTX_CLUSTER1 --set profile=default -y
      
      #Install Istio with custom configurations
      #Ingress Gateway with Privae IP Address
      #Another Publicly exposed LoadBalancer Service(L7) would be needed to access this Private IP
      istioctl install --context=$CTX_CLUSTER1 -f $istioPath/Components/values-primary.yaml -y
      
      #Following link describes - how to configure Application Gateway with AKS cluster
      #This need to be used if Ingress Gateway is having Private IP
      https://docs.microsoft.com/en-us/azure/application-gateway/configuration-http-settings 
      ```

      ##### Configure Istio in Primary Cluster

      - ##### Inject Istio into Namespaces

        ```bash
        #Inject Istio into primary namespace
        #Ensures sidecar container to be added for every deployment in this namespace
        kubectl label namespace primary istio-injection=enabled --context=$CTX_CLUSTER1
        
        #Disable sidecar injection from primary namespace
        #kubectl label namespace primary istio-injection=disabled --context=$CTX_CLUSTER1
        ```

      - ##### Install Addons

        ```bash
        #Install Istio Addons
        #This primarily installs all dependencies for observability by Istio viz. Grafana, Kiali dashboard etc.
        kubectl apply -f $istioPath/Components/samples/addons --context=$CTX_CLUSTER1
        
        #Check rollout status of the Kiali deployment - usually takes sometime
        kubectl rollout status deployment/kiali -n istio-system
        
        #Launch Kiali as background process
        istioctl dashboard kiali&
        
        #Might need to Press CTRL+C to allow the job to continue in teh background
        #Bring job to foreground - fg [<job-number>]
        
        #Check Deployments within istio-system
        #Istio Ingress gateway with public or private IP
        kubectl get svc -n istio-system
        ```

      - ##### Deploy BookInfo App

        ```bash
        #Install BookInfo app onto the cluster
        kubectl apply -f $istioPath/Examples/BookInfo/bookinfo.yaml -n primary
        
        #Check Deployed Components
        kubectl get svc -n primary --context=$CTX_CLUSTER1
        kubectl get pods -n primary --context=$CTX_CLUSTER1
        
        #Quick check to test BookInfo app
        podName=$(kubectl get pod -l app=ratings -n primary -o jsonpath='{.items[0].metadata.name}')
        kubectl exec $podName -n primary -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
        ```

      - ##### Expose Microservices thru an Ingress Gateway

        ```bash
        #Need a Gateway to expose the service outside
        #Check Routing definitions
        #Replace <dns-name>
        kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
        
        #Get GATEWAY_URL
        kubectl get svc istio-ingressgateway -n istio-system
        export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
        export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
        
        #Call services using GATEWAY_URL
        export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
        echo "$GATEWAY_URL"
        
        #Try the follwoing URL in the Browser or do a cUrl
        curl http://$GATEWAY_URL/product
        ```

      - ##### Expose Kiali dashboard service thru a Gateway

        ```bash
        #Need a Gateway to expose the Kiali service outside
        #Check Routing definitions
        #Replace <dns-name>
        kubectl apply -f $istioPath/Examples/Networking/kiali-gateway.yaml -n istio-system --context=$CTX_CLUSTER1
        
        #Launch Kiali in the browser
        http://$INGRESS_HOST:$INGRESS_PORT/
        
        #If used with Applicagtion Gateway then use appropriate Host header in Http Settings
        ```

      - ##### Observability

        ![istio-metrics-graph](./Assets/istio-metrics-graph.png)

        ![istio-metrics-inbound](./Assets/istio-metrics-inbound.png)

        ![istio-metrics-inbound](./Assets/istio-metrics-oob.png)

        

      ##### Deploy more apps (Optional)

      ![ratings-api-general](./Assets/ratings-api-general.png)

      

      ![ratings-api-flow2](./Assets/ratings-api-flow2.png)

      

      ![ratings-web-flow2](./Assets/ratings-web-flow2.png)

      ```bash
      #Create aks-workshop-dev namespace
      kubectl create ns aks-workshop-dev --context=$CTX_CLUSTER1
      
      #Inject Istio into aks-workshop-dev namespace
      #Ensures sidecar container to be added for every deployment in this namespace
      kubectl label namespace aks-workshop-dev istio-injection=enabled --context=$CTX_CLUSTER1
      
      #Disable sidecar injection from aks-workshop-dev namespace
      #kubectl label namespace aks-workshop-dev istio-injection=disabled --context=$CTX_CLUSTER1
      
      #Deploy backend Mongo DB as container
      kubectl create ns db --context=$CTX_CLUSTER1
      
      helm repo add bitnami https://charts.bitnami.com/bitnami
      helm search repo bitnami
      
      helm install ratingsdb bitnami/mongodb -n db \
      --set auth.username=ratingsuser,auth.password=ratingspwd,auth.database=ratingsdb \
      --set controller.nodeSelector.agentpool=$sysNodePoolName \
      --set controller.defaultBackend.nodeSelector.agentpool=$sysNodePoolName
      
      #Remove backend Mongo DB container
      #helm uninstall ratingsdb
      
      #RatingsApi - Ratings API backend 
      
      #Clone/Fork/Download Souerce code
      https://github.com/monojit18/mslearn-aks-workshop-ratings-api.git
      
      #CD to the director where Dockerfile exists
      #This docker build but performed in a Cloud Agent(VM) by ACR
      az acr build -t $acrName.azurecr.io/ratings-api:v1.0.0 -r $acrName .
      
      kubectl create secret generic aks-workshop-mongo-secret -n aks-workshop-dev --context=$CTX_CLUSTER1 \
      --from-literal=MONGOCONNECTION="mongodb://ratingsuser:ratingspwd@ratingsdb-mongodb.db:27017/ratingsdb"
      
      #Change <acrName> in the $helmPath/ratingsapi-chart/values-dev.yaml
      #Change <agentpool> in the $helmPath/ratingsapi-chart/values-dev.yaml
      helm install ratingsapi-chart -n aks-workshop-dev $helmPath/ratingsapi-chart/ -f $helmPath/ratingsapi-chart/values-dev.yaml
      helm upgrade ratingsapi-chart -n aks-workshop-dev $helmPath/ratingsapi-chart/ -f $helmPath/ratingsapi-chart/values-dev.yaml
      
      #Remove RatinsgAPI app 
      #helm uninstall ratingsapi-chart -n aks-workshop-dev
      
      #RatingsWeb - Ratings App Frontend
      ===================================
      #Clone/Fork/Download Souerce code
      https://github.com/monojit18/mslearn-aks-workshop-ratings-web.git
      
      #CD to the director where Dockerfile exists
      #This docker build but performed in a Cloud Agent(VM) by ACR
      az acr build -t $acrName.azurecr.io/ratings-web:v1.0.0 -r $acrName .
      
      #Change <acrName> in the $helmPath/ratingsapi-chart/values-dev.yaml
      #Change <agentpool> in the $helmPath/ratingsapi-chart/values-dev.yaml
      helm install ratingsweb-chart -n aks-workshop-dev $helmPath/ratingsweb-chart/ -f $helmPath/ratingsweb-chart/values-dev.yaml
      helm upgrade ratingsweb-chart -n aks-workshop-dev $helmPath/ratingsweb-chart/ -f $helmPath/ratingsweb-chart/values-dev.yaml
      
      #helm uninstall ratingsweb-chart -n aks-workshop-dev
      
      #Need a Gateway to expose the service outside
      #Check Routing definitions
      kubectl apply -f $istioPath/Examples/Networking/ratingsweb-gateway.yaml -n aks-workshop-dev --context=$CTX_CLUSTER1
      
      #If used with Applicagtion Gateway then use appropriate Host header in Http Settings
      ```

      

      ##### Traffic Splitting

      ![k8s-service](./Assets/k8s-service.png)

      

      ![k8s-mirroring](./Assets/k8s-mirroring.png)

      

      

      ![istio-trafficplit](./Assets/istio-trafficplit.png)

      

      ![istio-trafficplit](./Assets/istio-mirroring.png)

      

      - ###### Gateway

        - Single *Gateway* can contain multiple hosts

        - Scoped to a *Namespace*

      - ###### Virtual Service

        - Single *Virtual Service* can be mapped with multiple *Gateways*

        - Single *Virtual Service* can contain multiple hosts

        - Scoped to a *Namespace*

      - ###### Destination Rule

        - Defines Policies to traffic post Routing

        - *LoadBalancing, Connection Pool, Outlier*

        - Scoped to a *Namespace*

          

      ```bash
      #Traffic Shifting
      kubectl apply -f $istioPath/Examples/HelloWorld/helloworld-app.yaml -n primary --context=$CTX_CLUSTER1
      kubectl get po -n primary --context=$CTX_CLUSTER1
      
      #Check Routing definitions
      kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Destination Rule
      kubectl apply -f $istioPath/Examples/Networking/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
      
      kubectl apply -f $istioPath/Examples/HelloWorld/helloworld-app-v2.yaml -n primary --context=$CTX_CLUSTER1
      kubectl get po -n primary --context=$CTX_CLUSTER1
      
      #Check Routing behaviour
      #UNCOMMENT: Test Traffic Shifting
      #Update Primary Gateway Routes - Change Traffic weight
      kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Destination Rule
      kubectl apply -f $istioPath/Examples/Networking/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Routing behaviour again
      ```

      

      ##### Fault Injection

      ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)

      

      ![istio-trafficplit](./Assets/istio-fault.png)

      

      - ###### Introduce Fault

      ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: ratingsfault-virtual-service
      spec:
        hosts:
        - ratings
        http:
      #Inject Fault
        - fault:
      #Introduce Delay
      #Product APIs would throw timeout error
            delay:
      #50% of the calls would be delayed by 7 seconds
              percentage:
                value: 50.0
      #Reduce delay to see change in behaviour of the app
              fixedDelay: 7s
          route:
          - destination:
              host: ratings
              subset: v1
      ```

      ```bash
      #Fault Injection
      #Deploy Fault in Ratinsg API
      kubectl apply -f $istioPath/Examples/Network/ratingsfault-virtual-service.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Comments in the file
      #Introduce Delay
      #Check Routing behaviour
      ```

      - ###### Introduce Fix

      ```bash
      #Introduce Fix
      kubectl apply -f $istioPath/Examples/Networking/reviewsfix-virtual-service.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Comments in the file
      #Check Routing behaviour
      ```

      ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: reviewsfix-virtual-service
      spec:
        hosts:
        - reviews
        http:    
        - route:
          - destination:
              host: reviews
              subset: v1
            weight: 50
      #Route 50% of the Traffic to v3
      #v3 has the fix with a timeout value of 2.5 seconds
          - destination:
              host: reviews
              subset: v3
            weight: 50
      ```

      ##### Blue/Green

      ![istio-trafficplit-bluegreen](./Assets/istio-trafficplit-bluegreen.png)

      ```bash
      #Blue/Green
      #Deploy PodInfo Blue
      kubectl apply -f $istioPath/Examples/BlueGreen/podinfo-blue.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Routing definitions
      kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Destination Rule
      kubectl apply -f $istioPath/Examples/Networking/podinfo-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
      
      #Deploy PodInfo green
      kubectl apply -f $istioPath/Examples/BlueGreen/podinfo-green.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Routing behaviour
      #UNCOMMENT: Test Blue/Green
      #Update Primary Gateway Routes - Change Traffic weight
      kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Destination Rule
      kubectl apply -f $istioPath/Examples/Networking/podinfo-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Routing behaviour again
      ```

      ##### Circuit Breaker

      ```bash
      #Circuit Breaker
      #Deploy HttpBin App
      kubectl apply -f $istioPath/Examples/HttpBin/httpbin.yaml -n primary --context=$CTX_CLUSTER1
      kubectl apply -f $istioPath/Examples/Networking/httpbin-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Deploy HttpBin Destination Rule
      kubectl apply -f $istioPath/Examples/Networking/httpbin-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
      
      #Check Routing behaviour
      #UNCOMMENT: Test Circuit Breaking
      kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
      
      #Generate Load - e.g. JMeter or Fortio or any other Laod Testing client
      
      #Deploy Fortio client
      kubectl apply -f $istioPath/Examples/HttpBin/sample-client/fortio-deploy.yaml -n primary --context=$CTX_CLUSTER1
      
      #Make calls from Fortio client
      export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
      kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
      
      #Check Routing behaviour
      #Observer many calls being failed and circuit is broken and joined automatically
      #Change parameters in the $istioPath/Examples/Networking/httpbin-destination-rule.yaml file
      #Play around and see the chnage in the behaviour
      ```

      - ###### Example - Destination Rule with Circuit Breaker configurations

      ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: DestinationRule
      metadata:
        name: httpbin
      spec:
        host: httpbin
        trafficPolicy:
          connectionPool:
            tcp:
              maxConnections: 1
            http:
              http1MaxPendingRequests: 1
              maxRequestsPerConnection: 1
          outlierDetection:
            consecutive5xxErrors: 1
            interval: 1s
            baseEjectionTime: 3m
            maxEjectionPercent: 100
      EOF
      ```

      - ###### Circuit Broken - Errors in response

      ![istio-metrics-trace1](./Assets/istio-circuit-break-1.png)

      

      - ###### Circuit Broken - More Errors in response

      ![istio-metrics-trace1](./Assets/istio-circuit-break-2.png)

      

      - ###### No need to Break the Circuit - All Success

      ![istio-metrics-trace1](./Assets/istio-circuit-break-3.png)

      

      ##### Distributed Tracing

      ![istio-metrics-trace1](./Assets/istio-metrics-trace1.png)

      ![istio-metrics-trace1](./Assets/istio-metrics-trace2.png)

      

      ##### Service Mirroring

      ![k8s-mirroring](./Assets/k8s-mirroring.png)

      

      ![k8s-mirroring-cross](./Assets/k8s-mirroring-cross.png)

      

      - ###### Create and Configure Secondary Cluster

        ```bash
        #Service Mirroring or Shadowing
        #Create Secondary Cluster - CLI or Portal
        export CTX_CLUSTER2=secondary
        
        #Connect to Public AKS Cluster with Primary Context
        az aks get-credentials -g $primaryResourceGroup -n $secondaryClusterName --context $CTX_CLUSTER2
        
        kubectl config use-context $CTX_CLUSTER2
        
        #Check Cluster Health - Secondary
        kubectl get no --context=$CTX_CLUSTER2
        kubectl get ns --context=$CTX_CLUSTER2
        kubectl create namespace istio-system --context $CTX_CLUSTER2
        kubectl create namespace secondary --context $CTX_CLUSTER2
        
        #Install Istio CLI
        #Select Default Istio Profile settings
        #Ingress Gateway with Public IP Address
        istioctl install --context=$CTX_CLUSTER2 --set profile=default -y
        
        #Install Istio with custom configurations
        #Ingress Gateway with Privae IP Address
        #Another Publicly exposed LoadBalancer Service(L7) would be needed to access this Private IP
        istioctl install --context=$CTX_CLUSTER2 -f $istioPath/Components/values-secondary.yaml -y
        
        #Inject Istio into Secondary namespace of the cluster 
        #This ensures sidecar container to be added for every deployment in this namespace
        kubectl label namespace secondary istio-injection=enabled --context=$CTX_CLUSTER2
        
        #Install Istio Addons
        #This primarily installs all dependencies for observability by Istio viz. Grafana, Kiali dashboard etc.
        kubectl apply -f $istioPath/Components/samples/addons --context=$CTX_CLUSTER2
        
        kubectl get svc istio-ingressgateway -n istio-system
        export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
        export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
        
        export GATEWAY_URL2=$INGRESS_HOST:$INGRESS_PORT
        echo "$GATEWAY_URL2"
        
        kubectl apply -f $istioPath/Examples/HelloWorld/helloworld-app-v2.yaml -n secondary --context=$CTX_CLUSTER2
        kubectl get po -n secondary --context=$CTX_CLUSTER2
        
        #Need a Gateway to expose the service outside
        #Check Routing definitions
        #Replace <dns-name>
        kubectl apply -f $istioPath/Examples/Networking/secondary-gateway.yaml -n secondary --context=$CTX_CLUSTER2
        
        #Destination Rule
        kubectl apply -f $istioPath/Examples/Networking/helloworld-v2-destination-rule.yaml -n secondary --context=$CTX_CLUSTER2
        
        kubectl get svc -n secondary --context=$CTX_CLUSTER2
        kubectl describe svc -n secondary --context=$CTX_CLUSTER2
        kubectl get svc -A --context=$CTX_CLUSTER2
        ```

      - ###### Configure Primary to Mirror services

        ```bash
        #Switch to the Primary Cluster
        kubectl config use-context $CTX_CLUSTER1
        
        #Check Routing definitions
        kubectl apply -f $istioPath/Examples/Networking/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
        
        #Deploy components so that Mirroring can work
        #Replace <dns-name>
        kubectl apply -f $istioPath/Examples/Networking/primary-serviceentry.yaml -n primary --context=$CTX_CLUSTER1
        
        #Destination Rule
        #Replace <dns-name>
        kubectl apply -f $istioPath/Examples/Networking/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
        
        kubectl get svc -n primary --context=$CTX_CLUSTER1
        kubectl describe svc -n primary --context=$CTX_CLUSTER1
        kubectl get svc -A --context=$CTX_CLUSTER1
        
        #Call helloworld-v1
        #Observe that all calls being replicated to helloworld-v2 of secondary cluster
        ```
        
      - ###### CleanUP
      
        ```bash
        #Cleanup
        #Uninstall Istio setup - primary cluster
        istioctl x uninstall --set profile=default --purge --context=$CTX_CLUSTER1
        kubectl delete namespace istio-system --context=$CTX_CLUSTER1
          
        #Uninstall Istio setup - secondary cluster
        istioctl x uninstall --set profile=default --purge --context=$CTX_CLUSTER2
        kubectl delete namespace istio-system --context=$CTX_CLUSTER2
        ```
    
  - #### Linkerd

     ![istio-arch](./Assets/linkerd-arch.png)

    

    - ##### Define CLI Variables

      ```bash
      linkerdResourceGroup="secondary-workshop-rg"
      linkerdClusterName="secondary-mesh-cluster"
      linkerdAcrName="scdmeshacr"
      linkerdIngressName="linkerd-ing"
      linkerdIngressNSName="$linkerdIngressName-ns"
      linkerdIngressDeployName=""
      linkerdSysNodePoolName="agentpool"
          helmPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/AKS/Helm"
          linkerdPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/Linkerd"
      ```
    
    - ##### Connect to Secondary Cluster

      ```bash
      #Connect to Secondary Cluster - CLI or Portal
      export CTX_CLUSTER2=secondary
      
      kubectl config use-context $CTX_CLUSTER
      ```
    
    - ##### Install Nginx Ingress
    
      ```bash
      #Install Nginx Ingress Controller
      #Create Ingress Namespace
      kubectl create namespace $linkerdIngressNSName
      kubectl label namespace $linkerdIngressNSName name=$linkerdIngressNSName
      
      #Install nginx using Helm
      helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      helm repo update
      
      #Install Nginx Ingress controller
      helm install $linkerdIngressName ingress-nginx/ingress-nginx --namespace $linkerdIngressNSName \
      --set controller.nodeSelector.agentpool=$linkerdSysNodePoolName \
      --set controller.defaultBackend.nodeSelector.agentpool=$linkerdSysNodePoolName \
      
      #UNCOMMENT: Install Nginx Ingress controller as an Internal LoadBalancer - Private IP Address
      #IMPORTANT: 
      #--set controller.service.loadBalancerIP=$backendIpAddress \
      #--set controller.service.annotations.'service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet'=$aksIngressSubnetName
      ```
    
    - ##### Deploy K8s Ingress
    
      ```bash
      helm create ingress-chart
          
      helm install  ingress-chart -n secondary $helmPath/ingress-chart/ -f $helmPath/ingress-chart/values-dev.yaml
      helm upgrade  ingress-chart -n secondary $helmPath/ingress-chart/ -f $helmPath/ingress-chart/values-dev.yaml
      
      #UnInstall ingress
      #helm uninstall ingress-chart -n secondary
      ```
    
    - ##### Download Linkerd

      ```bash
          export LINKERD2_VERSION=stable-2.10.0
          
          curl -sL https://run.linkerd.io/install | sh
          linkerd check --pre
          linkerd version
          
          linkerd install  | kubectl apply -f -
          linkerd check
          
          Install Viz for Linkerd
          =========================
          linkerd viz install | kubectl apply -f -
          linkerd check
          linkerd viz dashboard&
          
          Install Jaeger for Linkerd
          ===========================
          linkerd jaeger install | kubectl apply -f -
          linkerd jaeger check
          linkerd jaeger dashboard&
      ```
    
    - ##### Inject Linkerd
    
      ```bash
      Inject Linkerd into Ingress Cntroller
      =======================================
      kubectl -n $linkerdIngressNSName get deploy/$linkerdIngressDeployName -o yaml | linkerd inject --ingress - | kubectl apply -f -
      
      Inject Linkerd into Namespaces
      ================================
      kubectl get deploy -n secondary -o yaml | linkerd inject - | kubectl apply -f -
      linkerd viz dashboard&
      ```
    
    - ##### Traffic Splitting
    
      ![istio-trafficplit](./Assets/istio-trafficplit.png)
    
      
    
      ```bash
      k create ns emojivoto
      
      kubectl config set-context --current --namespace=emojivoto
      
      #Deploy Ingress 
      kubectl apply -f $linkerdPath/Examples/EmojiVoto/emojivoto-ingress.yaml 
      kubectl apply -f $linkerdPath/Examples/EmojiVoto/emojivoto.yaml
      
      #Inject Linkerd into emojivoto namespace
      kubectl get deploy -n emojivoto -o yaml | linkerd inject - | kubectl apply -f -
      
      #Display Linkerd dashboard as a background service
      linkerd viz dashboard&
      
      #Split Traffic
      kubectl apply -f $linkerdPath/Examples/EmojiVoto/emojivoto-traffic-split.yaml
      
      #UNCOMMENT: Test Traffic Shifting
      #Modify Traffic weight
      kubectl apply -f $linkerdPath/Examples/EmojiVoto/emojivoto-traffic-split.yaml
      
      #Check Routing behaviour
      
      ```
    
      - ###### Example - Split Traffic
      
        ```yaml
        apiVersion: split.smi-spec.io/v1alpha1
        kind: TrafficSplit
        metadata:
          name: emojivoto-split
          namespace: emojivoto
        spec:  
          service: web-svc
          backends:
          - service: web-svc
            weight: 100
          
        #UNCOMMENT: Test Traffic Shifting
          # - service: web-svc-v12
          #   weight: 0
        ```
      
      
    ![linkerd-traffic-split](./Assets/linkerd-traffic-split.png)      
      
    
    
    - ##### Blue/Green
    
      ```bash
      k create ns test
      
      kubectl config set-context --current --namespace=test
      
      #Deploy frontend-blue app
      kubectl apply -f $linkerdPath/Examples/BlueGreen/frontend-blue.yaml
      
      #Deploy frontend-green app
      kubectl apply -f $linkerdPath/Examples/BlueGreen/frontend-green.yaml
      
      #Deploy Ingress 
      kubectl apply -f $linkerdPath/Examples/BlueGreen/frontend-ingress.yaml
      
      #Check Comments
      
      #Inject Linkerd into test namespace
      kubectl get deploy -n test -o yaml | linkerd inject - | kubectl apply -f -
      
      #Display Linkerd dashboard as a background service
      linkerd viz dashboard&
      
      #Split Traffic
      kubectl apply -f $linkerdPath/Examples/BlueGreen/frontend-traffic-split.yaml
      
      #UNCOMMENT: Test Traffic Shifting
      #Modify Traffic weight
      kubectl apply -f $linkerdPath/Examples/EmojiVoto/frontend-traffic-split.yaml
      
      #Check Routing behaviour
      ```

    - ##### Deploy more apps - Ratings app (Optional)
    
      ```bash
      #Deploy backend DB as container
      kubectl create ns db --context=$CTX_CLUSTER1
      
      helm repo add bitnami https://charts.bitnami.com/bitnami
      helm search repo bitnami
      
      helm install ratingsdb bitnami/mongodb:4.4.10 -n db \
      --set auth.username=ratingsuser,auth.password=ratingspwd,auth.database=ratingsdb \
      --set controller.nodeSelector.agentpool=agentpool \
      --set controller.defaultBackend.nodeSelector.agentpool=agentpool
      ```
    
    - ##### Observability
    
      - ###### View Basic Metrics
      
        ![linkerd-grafana1](./Assets/linkerd-grafana1.png)
      
        ![linkerd-grafana2](./Assets/linkerd-grafana2.png)
        
        
    
    - ##### Distributed Tracing
    
      ![linkerd-jaeger](./Assets/linkerd-jaeger.png)
    
      ```bash
      kubectl -n emojivoto set env --all deploy OC_AGENT_HOST=collector.linkerd-jaeger:55678
      for ((i=1;i<=100;i++)); do   curl -kubectl "https://emojivoto.domain.com/api/list"; done
      ```
    
    - ##### Fault Injection
    
      ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)
    
      ```bash
      #Left as an Exercise
      ```
    
    - ##### Circuit Breaking
    
      ```bash
      #Left as an Exercise
      ```
    
    - ##### Service Mirroring
    
      ![linkerd-mirroring](./Assets/linkerd-mirroring.png)
    
      ```bash
      #Left as an Exercise  
      ```

  - ### API Mesh

    ![apimesh](./Assets/apimesh.png)

    - Authentication thru OAuth or OIDC

    - Deep integration with Azure AD B2B and B2C scenarios

    - APIM Policies

      - Caching
      - Rate Limiting
      - Retry
      - URL Rewrite
      - Response/Response body transformation e.g. JSON to XML and vice versa
      - Control Flow and Usage
      - **Custom Role, RBAC**
      - .....

    ```bash
    #Declare Local variables for APIM
    apimeshPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/APIMesh"
    
    #Deploy APIM on AKS
    Ref: https://docs.microsoft.com/en-us/azure/api-management/how-to-deploy-self-hosted-gateway-kubernetes
    kubectl create secret generic apimesh-token -n apimesh --from-literal=value="GatewayKey apimesh&202112041054&73uoRfRgH4qcMmDvNzaL4dKRCoFI9IbCYhkJzrrvl1aVAaW5+bCghCsLDdEnMWbQJHiOkzUWzt093ocZRfb6BA=="  --type=Opaque
    kubectl apply -f $apimeshPath/apimesh.yaml -n apimesh
    
    
    #Expose APIM as Internal LoadBalancer Service
    #Check Comments
    Modify $apimeshPath/apimesh.yaml
    
    #Public facing L7 LoadBalancer like Application Gateway would call this Service directly
    #Define RatingsAPI (deloyed earlier) behind APIM
    #Test API calls through Application Gateway
    ```

    - **Example - APIM Service** 

      ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: apimesh
        annotations:
          service.beta.kubernetes.io/azure-load-balancer-internal: "true"
          service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "secondary-ingress-subnet"
      spec:
      #Define type as LoadBalancer
        type: LoadBalancer
      #Assign a fixed Private IP  
        loadBalancerIP: 31.0.5.100
        ports:
        - name: http
          port: 80
          targetPort: 8080
        - name: https
          port: 443
          targetPort: 8081
        selector:
          app: apimesh
      ```

    - **Define APIs behind APIM; running withon AKS cluster**

      ![istio-mirroring-2](./Assets/apimesh-1.png)

      ![istio-mirroring-2](./Assets/apimesh-2.png)

      ![istio-mirroring-2](./Assets/apimesh-3.png)

      

- ## What Next

  - ### Multi Cluster

    ![istio-mirroring-2](./Assets/istio-mirroring-2.png)

    ![istio-mirroring-2](./Assets/istio-mirroring-3.png)

    - ### Unified Model across Clusters
      
      - Observability
      
      - Reliability
      
      - Trust/Security
      
      - Separate Failure domain - Faliure of one cluster transfers control to Others
      
      - Supporting different/heteropgeneous networks
      
        

  - ### Extended Elastic Scaling

    - Custom solution using K8s REST API
    - Monitor Cluster Metric and Scale Applications across Multiple clusters
    - On-Prem to Cloud Scaling on need basis

  

- ## References

  - **[Istio](https://istio.io/latest/docs/setup/getting-started/)**
  - **[Linkerd](https://linkerd.io/2.11/getting-started/)**
  - **[Open Service Mesh](https://openservicemesh.io/)**
    - [About](https://docs.microsoft.com/en-us/azure/aks/open-service-mesh-about)
  - **[APIM Self Hosted](https://docs.microsoft.com/en-us/azure/api-management/self-hosted-gateway-overview)**
  - **[Github Repo](https://github.com/monojit18/ServiceMeshWorkshop)**

