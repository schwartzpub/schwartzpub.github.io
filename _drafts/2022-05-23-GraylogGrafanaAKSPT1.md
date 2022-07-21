---
title: Migrating Graylog and Grafana to Azure Kubernetes Service - Part 1
categories:
  - blog
tags:
  - Jekyll
  - update
toc: true

---
# Part 1 - Set up Base Infrastructure
## Step 1 - Create a new Resource Group
## Step 2 - Create a new (Private) AKS Cluster
## Step 3 - Create a new (Private) AKS Container Registry
### Step 3.1 Create a private endpoint
### Step 3.2 Add zones to DNS
## Step 4 - Create a new AKS Secret Vault
## Step 5 - Create a new Node Pool
## Step 6 - Graylog, MongoDB, Elasticsearch
### Step 6.1 Create the underlying persistent storage
    - Create Disks, Persistent Volume Claims, Persisitent Volumes
### Step 6.2 Create a Standard Internal Load Balancer (service)
    - Create Outbound NAT gateway
