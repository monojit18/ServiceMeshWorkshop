# Connected Micro services with K8s and ServiceMesh



## Introduction

## Purpose

## What to Accomplish

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

  

  - ## HOL

    ![workflow](./Assets/workflow.png)

    

    - ### Let us Delve into it

      - #### **Clone Workshop repo**

        ```bash
        git clone https://github.com/monojit18/ServiceMeshWorkshop
        ```

      - #### Plan Workshop Tasks

        - Define Local variables

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
          aksVnetPrefix="31.0.0.0/21"
          aksVnetId=
          aksSubnetName="primary-cluster-subnet"
          aksSubnetPrefix="31.0.0.0/24"
          aksSubnetId=
          aksServicePrefix="31.0.1.0/24"
          dnsServiceIP="31.0.1.10"
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
          
        - Login to Azure
        
          ```bash
          #Login to Azure
          az login --tenant $tenantId
          
          #Check Selected Subscription
          az account show
          
          #Set appropriate Subscription, if needed
          #az account set -s $subscriptionId
          ```
        
        - Create Resource Group
        
          ```bash
          #Create Resource Group for AKS workloads
          az group create -n $aksResourceGroup -l $location
          ```

        - Service Principal
        
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

        - ##### **Create Virtual Network**

          ```bash
          #Deploy Virtual Network
          az network vnet create -n $aksVnetName -g $aksResourceGroup --address-prefixes $aksVnetPrefix
          aksVnetId=$(az network vnet show -n $aksVnetName -g $aksResourceGroup --query="id" -o tsv)
          echo $aksVnetId
          ```

          

        - ##### **Create Subnet for AKS cluster**

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

        

        

        - ##### Configure AKS Cluster - Primary

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

        - ##### Set CLI Variables for Istio

          ```bash
          primaryResourceGroup=$aksResourceGroup
          primaryClusterName=$clusterName
          secondaryResourceGroup="secondary-workshop-rg"
          secondaryClusterName="secondary-mesh-cluster"
          primaryAcrName=$acrName
          helmPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/AKS/Helm"
          istioPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/Istio"
          ```

        - ##### Download Istio

          ```bash
          #Download Istio binary
          curl -L https://istio.io/downloadIstio | sh -
          
          #Download specific version of Istio viz. 1.11.3
          curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.11.3 TARGET_ARCH=x86_64 sh -
          
          #The istioctl client binary in the bin/ directory
          #Add the istioctl client to your path (Linux or macOS):
          export PATH=$PWD/bin:$PATH
          ```

        - ##### Check Cluster Health

          ```bash
          kubectl get no --context=$CTX_CLUSTER1
          kubectl get ns --context=$CTX_CLUSTER1
          ```
          
        - ##### Create Namespaces

          ```bash
          #Create namespaces for Istio
          kubectl create namespace istio-system --context $CTX_CLUSTER1
          kubectl create namespace primary --context $CTX_CLUSTER1
          ```

        - ##### Install Istio CLI

          ```bash
          #Select Default Istio Profile settings
          istioctl install --context=$CTX_CLUSTER1 --set profile=default -y
          ```

        - ##### Configure Istio in Primary Cluster

          - ###### Inject Istio into Namespaces

            ```bash
            #Inject Istio into Primary namespace of the cluster 
            #This ensures sidecar container to be added for every dpeloyment in this namespace
            kubectl label namespace primary istio-injection=enabled --context=$CTX_CLUSTER1
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
            #Istio Ingress gateway with public IP
            kubectl get svc -A
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

            ```bash
            #Need a Gateway to expose the service outside
            #Check Routing definitions
            kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
            
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

          - ##### Observability

            ![istio-metrics-graph](./Assets/istio-metrics-graph.png)

            ![istio-metrics-inbound](./Assets/istio-metrics-inbound.png)

            ![istio-metrics-inbound](./Assets/istio-metrics-oob.png)

            ###### 

        - ##### Deploy more apps (Optional)

          ![ratings-api-general](./Assets/ratings-api-general.png)

          

          ![ratings-api-flow2](./Assets/ratings-api-flow2.png)

          

          ![ratings-web-flow2](./Assets/ratings-web-flow2.png)

          ```bash
          #Create namespace
          kubectl create ns aks-workshop-dev --context=$CTX_CLUSTER1
          
          #Inject Istio into namespace
          #This ensures sidecar container to be added for every dpeloyment in this namespace
          kubectl label namespace aks-workshop-dev istio-injection=enabled --context=$CTX_CLUSTER1
          
          #Deploy backend DB as container
          kubectl create ns db --context=$CTX_CLUSTER1
          
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm search repo bitnami
          
          helm install ratingsdb bitnami/mongodb -n db \
          --set auth.username=ratingsuser,auth.password=ratingspwd,auth.database=ratingsdb \
          --set controller.nodeSelector.agentpool=$sysNodePoolName \
          --set controller.defaultBackend.nodeSelector.agentpool=$sysNodePoolName
          
          
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
          kubectl apply -f $istioPath/Examples/Gateways/ratings-gateway.yaml -n aks-workshop-dev --context=$CTX_CLUSTER1
          
          #Check Routing behaviour
          #UnComment follwoing line in $istioPath/Examples/Gateways/primary-gateway.yaml
          /*
           # - uri:
              #     prefix: /static
          */
          #This would make RatingsWeb app to fail
          #Check Routing behaviour again
          ```

          

        - ###### Traffic Splitting

          ![k8s-service](./Assets/k8s-service.png)

          

          ![k8s-mirroring](./Assets/k8s-mirroring.png)

          

          

          ![istio-trafficplit](./Assets/istio-trafficplit.png)

          ```bash
          #Traffic Shifting
          kubectl apply -f $istioPath/Examples/HelloWorld/helloworld-app.yaml -n primary --context=$CTX_CLUSTER1
          kubectl get po -n primary --context=$CTX_CLUSTER1
          
          #Check Routing definitions
          kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
          #Destination Rule
          kubectl apply -f $istioPath/Examples/Gateways/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
          
          #Check Routing behaviour
          #Update Primary Gateway Routes - Change Traffic weight
          
          kubectl apply -f $istioPath/Examples/HelloWorld/helloworld-app-v2.yaml -n primary --context=$CTX_CLUSTER1
          kubectl get po -n primary --context=$CTX_CLUSTER1
          
          #Check Routing definitions
          kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
          #Destination Rule
          kubectl apply -f $istioPath/Examples/Gateways/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
          
          #Check Routing behaviour again
          ```

          

        - ###### Fault Injection

          ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)

          ```yaml
          apiVersion: networking.istio.io/v1alpha3
          kind: Gateway
          metadata:
            name: ratingsweb-gateway  
          spec:
            selector:
              istio: ingressgateway # use istio default controller
            servers:
            - port:
                number: 80
                name: http
                protocol: HTTP
              hosts:
              - "*"
          ---
          apiVersion: networking.istio.io/v1alpha3
          kind: VirtualService
          metadata:
            name: ratingsweb-virtual-service  
          spec:
            hosts:
            - "*"
            gateways:
            - ratingsweb-gateway
            http:
            # - fault:
            #     delay:
            #       fixedDelay: 3s
            #       percentage:
            #         value: 100
            #   route:
            #   - destination:
            #       host: ratingsweb-service
            #       port:
            #         number: 80  
            - match:
              - uri:
                  regex: /?(.*)  
              route:
              - destination:
                  host: ratingsweb-service
                  port:
                    number: 80
          ```

          ```bash
          #Check Routing definitions
          kubectl apply -f $istioPath/Examples/Gateways/ratings-gateway.yaml -n primary --context=$CTX_CLUSTER1
          
          #UnComment Fault section
          kubectl apply -f $istioPath/Examples/Gateways/ratings-gateway.yaml -n primary --context=$CTX_CLUSTER1
          #Check Routing definitions again
          ```

          

        - ###### Distributed Tracing

          ![istio-metrics-trace1](./Assets/istio-metrics-trace1.png)

          ![istio-metrics-trace1](./Assets/istio-metrics-trace2.png)

        - ###### Blue/Green

          ```bash
          #Blue/Green
          #Deploy PodInfo Blue
          kubectl apply -f $istioPath/Examples/BlueGreen/podinfo-blue.yaml -n primary --context=$CTX_CLUSTER1
          
          #Check Routing definitions
          kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
          #Destination Rule
          kubectl apply -f $istioPath/Examples/Gateways/podinfo-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
          
          #Deploy PodInfo green
          kubectl apply -f $istioPath/Examples/BlueGreen/podinfo-green.yaml -n primary --context=$CTX_CLUSTER1
          
          #Update Primary Gateway Routes - Change Traffic weight
          #Check Routing definitions
          kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
          #Destination Rule
          kubectl apply -f $istioPath/Examples/Gateways/podinfo-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
          
          #Check Routing behaviour again
          ```

        - ###### Circuit Breaker

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

          ```bash
          #Circuit Breaker
          #Deploy HttpBin App
          kubectl apply -f $istioPath/Examples/HttpBin/httpbin.yaml -n primary --context=$CTX_CLUSTER1
          kubectl apply -f $istioPath/Examples/Gateways/httpbin-gateway.yaml -n primary --context=$CTX_CLUSTER1
          
          #Deploy HttpBin Destination Rule
          kubectl apply -f $istioPath/Examples/Gateways/httpbin-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
          
          #Deploy Fortio client
          kubectl apply -f $istioPath/Examples/HttpBin/sample-client/fortio-deploy.yaml -n primary --context=$CTX_CLUSTER1
          
          #Make class from Fortio client
          export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
          kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
          
          #Check Routing behaviour
          ```

          

        - ###### Service Mirroring

          

          ![k8s-mirroring](./Assets/k8s-mirroring.png)

          

          ![k8s-mirroring-cross](./Assets/k8s-mirroring-cross.png)

          

          ![istio-trafficplit](./Assets/istio-mirroring.png)

          

          - ###### Create and Configure Secondary Cluster

            ```bash
            #Create Secondary Cluster - CLI or Portal
            export CTX_CLUSTER2=secondary
            
            #Connect to Public AKS Cluster with Primary Context
            az aks get-credentials -g $secondaryResourceGroup -n $secondaryClusterName --context $CTX_CLUSTER2
            
            kubectl config use-context $CTX_CLUSTER2
            
            #Check Cluster Health - Secondary
            kubectl get no --context=$CTX_CLUSTER2
            kubectl get ns --context=$CTX_CLUSTER2
            kubectl create namespace istio-system --context $CTX_CLUSTER2
            kubectl create namespace secondary --context $CTX_CLUSTER2
            
            #Install Istio CLI
            ##Select Default Istio Profile settings
            istioctl install --context=$CTX_CLUSTER2 --set profile=default -y
            
            #Inject Istio into Secondary namespace of the cluster 
            #This ensures sidecar container to be added for every dpeloyment in this namespace
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
            
            kubectl apply -f $istioPath/Examples/Gateways/helloworld-v2-gateway.yaml -n secondary --context=$CTX_CLUSTER2
            kubectl apply -f $istioPath/Examples/Gateways/helloworld-v2-destination-rule.yaml -n secondary --context=$CTX_CLUSTER2
            
            kubectl get svc -n secondary --context=$CTX_CLUSTER2
            kubectl describe svc -n secondary --context=$CTX_CLUSTER2
            kubectl get svc -A --context=$CTX_CLUSTER2
            ```

          - ###### Configure Primary to Mirror services

            ```bash
            #Switch to the Primary Cluster
            kubectl config use-context $CTX_CLUSTER1
            
            #Check Routing definitions
            kubectl apply -f $istioPath/Examples/Gateways/primary-gateway.yaml -n primary --context=$CTX_CLUSTER1
            
            #Deploy components so that Mirroring can work
            kubectl apply -f $istioPath/Examples/Gateways/helloworld-serviceentry.yaml -n primary --context=$CTX_CLUSTER1
            kubectl apply -f $istioPath/Examples/Gateways/helloworld-destination-rule.yaml -n primary --context=$CTX_CLUSTER1
            
            kubectl get svc -n primary --context=$CTX_CLUSTER1
            kubectl describe svc -n primary --context=$CTX_CLUSTER1
            kubectl get svc -A --context=$CTX_CLUSTER1
            ```

          - ###### CleanUp

            ```bash
            #Uninstall Istio setup
            istioctl x uninstall --set profile=default --purge --context=$CTX_CLUSTER1
            kubectl delete namespace istio-system --context=$CTX_CLUSTER1
            
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
          $linkerdIngressName="linkerd-ing"
          $linkerdIngressNSName="$linkerdIngressName-ns"
          $linkerdIngressDeployName=""
          helmPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/AKS/Helm"
          $linkerdPath="/Users/monojitdattams/Development/Projects/Workshops/AKSWorkshop/ServiceMeshWorkshop/Linkerd"
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
          
          #Install Ingress controller
          helm install $linkerdIngressName ingress-nginx/ingress-nginx --namespace $linkerdIngressNSName \
          #--set controller.service.loadBalancerIP=$backendIpAddress \
          #--set controller.service.annotations.'service\.beta\.kubernetes\.io/azure-load-balancer-internal-subnet'=$aksIngressSubnetName \
          --set controller.nodeSelector.agentpool=$sysNodePoolName \
          --set controller.defaultBackend.nodeSelector.agentpool=$sysNodePoolName
          ```

        - ##### Deploy K8s Ingress

          ```bash
          helm create ingress-chart
          
          helm install  ingress-chart -n secondary $helmPath/ingress-chart/ -f $helmPath/ingress-chart/values-dev.yaml
          helm upgrade  ingress-chart -n secondary $helmPath/ingress-chart/ -f $helmPath/ingress-chart/values-dev.yaml
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
          ```

        - ##### Observability

          - ###### View Basic Metrics

            ![linkerd-grafana1](./Assets/linkerd-grafana1.png)

            ![linkerd-grafana2](./Assets/linkerd-grafana2.png)

            

        - ###### Traffic Splitting

          ![istio-trafficplit](./Assets/istio-trafficplit.png)

          

          ```bash
          kubectl config set-context --current --namespace=emojivoto
          kubectl apply -f $linkerdPath/Examples/emojivoto-ingress.yaml 
          kubectl apply -f $linkerdPath/Examples/emojivoto.yaml
          kubectl get deploy -n emojivoto -o yaml | linkerd inject - | kubectl apply -f -
          ```

          ![linkerd-traffic-split](./Assets/linkerd-traffic-split.png)

          

        - ###### Fault Injection

          ![istio-trafficplit](./Assets/istio-trafficplit-faultinjection.png)

          

          ```bash
          kubectl -n emojivoto set env --all deploy OC_AGENT_HOST=collector.linkerd-jaeger:55678
          for ((i=1;i<=100;i++)); do   curl -kubectl "https://emojivoto.domain.com/api/list"; done
          ```

        - ###### Deploy more apps - Ratings app (Optional)

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

          

        - ###### Circuit Breaking

          ```bash
          #Left as an Exercise
          ```

          

        - ###### Distributed Tracing

          ![linkerd-jaeger](./Assets/linkerd-jaeger.png)
          
          

        - ###### Service Mirroring

          ![linkerd-mirroring](./Assets/linkerd-mirroring.png)
          
          ```bash
          #Left as an Exercise
          ```

  - ## What Next

    - ### Multi Cluster

      ![istio-mirroring-2](./Assets/istio-mirroring-2.png)

      ![istio-mirroring-2](./Assets/istio-mirroring-3.png)

      

    - ### API Mesh

      ![apimesh](./Assets/apimesh.png)

      

  - ## References

    - Istio - https://istio.io/latest/docs/setup/getting-started/
    - Linkerd - https://linkerd.io/2.11/getting-started/
    - APIM Self Hosted - https://docs.microsoft.com/en-us/azure/api-management/self-hosted-gateway-overview
    - Github Repo - 
