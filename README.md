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

    

    - #### Service Mesh

      - ##### Istio

        - ###### Download Istio

        - ###### Install Istio CLI - istioctl

        - ###### Configure Istio in Primary Cluster

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

      - ##### Linkerd

        - ###### Download Linkerd

        - ###### Install Linkerd CLI - linkerdctl

        - ###### Configure Linkerd in Primary Cluster

          - ###### Install Linkerd onto Primary Cluster

          - ###### Inject Linkerd into Namespaces

          - ###### Install Viz dashboard

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

            

  - ## References
