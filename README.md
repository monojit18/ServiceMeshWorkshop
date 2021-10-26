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

  - #### Create AKS Cluster

    - ##### Create Service Principal

    - ##### Create Network for AKS

      - ###### Create Virtual Network

      - ###### Create Subnet for AKS cluster

      - ###### Create Subnet for Ingress Controller

    - ##### Assign Roles to Service Principal

    - ##### Create Azure Container Registry (ACR)

    - ##### Create AKS cluster

  - #### Cluster Configuration - Post Creation

    - ##### Create Namespaces

      - DEV
      - QA
      - Smoke

    - ##### Deploy RBAC

    - ##### Configure SSL (*Optional*)

      - ###### Get SSL Certificate

      - ###### Create K8s Secret

    - ##### Install Nginx Ingress Controller

    - ##### Deploy Ingress

      - ###### Configure Ingress with SSL

      - ###### Configure Routes

  - #### Deploy Microservices

    - ##### Deploy MongoDB Container

    - ##### Deploy Ratings Web app

    - ##### Deploy Ratings Api app

  - Test 

  ### 

- ### Maintenance
