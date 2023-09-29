
# Monitoring of a K8S Multi-Cluster architecture Using Prometheus and Grafana

i m going to explain the process of the provision of a K8S multi-cluster architecutre on Azure and monitor it using Prometheus and Grafana

### Architecture Used

So for this Project we will adopt the following architecture

![](https://cdn-images-1.medium.com/max/2000/1*nSIHaHFRN2xyyHWDRuQQDA.png)

A quick Overview for this architecture is that i m going to provision 2 k8s clusters on AKS (Azure Kubernetes Service) then i m going to install prometheus node exporter inside each cluster once the provision of the 2 clusters and the configuration is done i m going to create an azure virtual machine (ubuntu server) on which i m going to deploy prometheus server and configure it in a way that this server will scrape data from the 2 nodes-exporter deployed inside each cluster and then i will just install grafana and use it to draw dashboards to visualise the behaviour of our clusters

### Tools Used

The provision of the 2 cluster was made by Terraform , The deployment of prometheus node exporters i did it by the help a jenkins pipeline , and the configuration of the virtual machine was manually made

### Provision of the Clusters :

As mentioned in the previous part i m going to provision the clusters using Terraform

Well Terraform is an infrastructure as code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share

So as it is based on configuration files i write the following file to provision my 2 clusters

    terraform {
      required_providers {
        azurerm = {
          source  = "hashicorp/azurerm"
          version = "3.54.0"
        }
      }
      required_version = ">= 0.14.9"
    }
    
    provider "azurerm" {
      features {}
      subscription_id = var.subscription_id
    }
    resource "azurerm_resource_group" "resource-group" {
      name     = "AKS-resource-group"
      location = "East US"
      
    }
    resource "azurerm_kubernetes_cluster" "cluster1" {
      name                = "cluster1-aks"
      location            = azurerm_resource_group.resource-group.location
      resource_group_name = azurerm_resource_group.resource-group.name
      dns_prefix          = "cluster1-k8s-dns"
      kubernetes_version  = "1.26.3"
    
      default_node_pool {
        name            = "default"
        node_count      = 1
        vm_size         = "Standard_A2_v2"
        os_disk_size_gb = 30
      }
    
     identity {
        type = "SystemAssigned"
      }
      tags = {
        environment = "Demo"
      }
    }
    resource "azurerm_kubernetes_cluster" "cluster2" {
      name                = "cluster2-aks"
      location            = azurerm_resource_group.resource-group.location
      resource_group_name = azurerm_resource_group.resource-group.name
      dns_prefix          = "cluster2-k8s-dns"
      kubernetes_version  = "1.26.3"
    
      default_node_pool {
        name            = "default"
        node_count      = 1
        vm_size         = "Standard_A2_v2"
        os_disk_size_gb = 30
      }
    
     identity {
        type = "SystemAssigned"
      }
      tags = {
        environment = "Demo"
      }
    }

Ok here i just created a resource group and i added 2 clusters inside it with 1 node and a standard_A2_v2 vm_size so that my node take only 2vcpu

i apply this configuration by terraform apply

![](https://cdn-images-1.medium.com/max/2704/1*Qv_cx3FtC2exNh-R7jX4Gg.png)

and once the provision is done

![](https://cdn-images-1.medium.com/max/2000/1*_oEAmgVJP_vn7Gr1AXolZA.png)

We can simply check the existence on Azure

![](https://cdn-images-1.medium.com/max/2724/1*W-pUS8OdlDb2Hs9JudsETQ.png)

### Prometheus

![](https://cdn-images-1.medium.com/max/2000/0*rh4lF2bYgMJmYbnU.png)

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.

Prometheus collects and stores its metrics as time series data, which means that metrics information is stored with the timestamp at which it was recorded, alongside optional key-value pairs called labels.

To get metrics, Prometheus requires an exposed HTTP endpoint. Once an endpoint is available, Prometheus can start scraping numerical data, capture it as a time series, and store it in a local database suited to time-series data. Prometheus can also be integrated with remote storage repositories.

### Deployment of the Prometheus

I defined this Pipeline

    pipeline{
        agent any
        stages{
            stage("getting code") {
                steps {
                    git url: 'https://github.com/Louaykharouf26/Multi-Cluster-Monitoring.git', branch: 'master',
                    credentialsId: 'github-credentials' //jenkins-github-creds
                    sh "ls -ltr"
                }
            }
           stage("Setting up cluster 1") {
                steps {                
                    script {
                        echo "======== executing ========"
                            sh "pwd"
                            sh "az login"
                            sh "az aks get-credentials --resource-group AKS-resource-group --name cluster1-aks"
                            sh "kubectl apply -f https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3"
                            echo "installing prometheus using helm"
                            sh "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts"
                            sh "helm install prometheus prometheus-community/prometheus"
                            sh "kubectl get pods"
                             }            
                            }
                        } 
                    stage("Setting up cluster 2") {
                steps {                
                    script {
                        echo "======== executing ========"
                            sh "pwd"
                            sh "az login"
                            sh "az aks get-credentials --resource-group AKS-resource-group --name cluster2-aks"
                            sh "kubectl apply -f https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3"
                            echo "installing prometheus using helm"
                            sh "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts"
                            sh "helm install prometheus prometheus-community/prometheus"
                            sh "kubectl get pods"
                             }            
                            }
                        } 
                    }
                post{
                    success{
                        echo "======== Setting up infra executed successfully ========"
                    }
                    failure{
                        echo "======== Setting up infra execution failed ========"
                    }
                }
                 
            }          
       /* 
        post{
            success{
                echo "========pipeline executed successfully ========"
            }
            failure{
                echo "========pipeline execution failed========"
            }
        }*/

Through this pipeline i will just install helm inside each cluster and then with the help of helm i will deploy the prometheus inside each cluster

Helm is a tool that streamlines installing and managing Kubernetes applications

the execution will look like this

![](https://cdn-images-1.medium.com/max/2000/1*MT7p3LNttTtPNyI15LXRZw.png)

and with the success of our pipeline now we have prometheus installed succesfully inside our clusters

Ok now lets keep our clusters and focus on the preparation of prometheus server and grafana on an azure virtual machine

### Grafana And Prometheus Configuration

To install and configure prometheus server and grafana i provisioned this virtual machine

![](https://cdn-images-1.medium.com/max/2712/1*GKbQpCDxS4XWT2yFfyG7qQ.png)

Before i start here is a quick overview of what is grafana about

![](https://cdn-images-1.medium.com/max/2000/0*PaHYrXjcb1k1sY3j.png)

Grafana is an open source interactive data-visualization platform, developed by Grafana Labs, which allows users to see their data via charts and graphs that are unified into one dashboard (or multiple dashboards!) for easier interpretation and understanding. You can also query and set alerts on your information and metrics from wherever that information is stored, whether that’s traditional server environments, Kubernetes clusters, or various cloud services, etc.

ok with a quick ssh to our vm we can start the installation by installing prometheus by running this command

    curl -LO https://github.com/prometheus/prometheus/releases/download/v2.0.0/prometheus-2.0.0.linux-amd64.tar.gz
    tar xvf prometheus-2.0.0.linux-amd64.tar.gz
    sudo cp -r prometheus-2.0.0.linux-amd64/consoles /etc/prometheus
    sudo cp -r prometheus-2.0.0.linux-amd64/console_libraries /etc/prometheus

Ok now it is installed previously in the step of the deployment of prometheus in the cluster we had defined the service of the node exporter and from the azure portal i can get the endpoint of the 2 node-exporters of the cluster

![](https://cdn-images-1.medium.com/max/2000/1*qp2wY8mJjG9duUVLKStNQg.png)

![](https://cdn-images-1.medium.com/max/2000/1*558wbEcKc4hOAYqD9OXRAg.png)

ok now lets configure our prometheus server to scrape data from this 2 nodes this will be done through a prometheus.yml file that contains the endpoints that prometheus will use to scrape data for the metrics that the user needs

the configuration file will look like this

![](https://cdn-images-1.medium.com/max/2000/1*5VWqRYj12Gq_1pZ-O5r4Fg.png)

ok here prometheus will scrape data from the endpoints and even from the local virtual machine

lets now apply this configuration

    sudo /usr/local/bin/prometheus \
        --config.file /etc/prometheus/prometheus.yml \
        --storage.tsdb.path /var/lib/prometheus/ \
        --web.console.templates=/etc/prometheus/consoles \
        --web.console.libraries=/etc/prometheus/console_libraries

and we can check from the browser that the node-exporters are succefully linked to our prometheus server

![](https://cdn-images-1.medium.com/max/2732/1*rjzLplWTd6Q0dje1uir0wA.png)

and here all our node exporters are UP and prometheus is scrapping succefully data

lets move now to the configuration of grafana

lets install it first on the VM

    wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
    sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
    sudo apt update
    sudo apt install grafana
    sudo systemctl start grafana-server
    sudo systemctl enable grafana-server

now let’s open it through our browser and connect using admin admin and set our password

the UI will look like this once logged in

![](https://cdn-images-1.medium.com/max/2732/1*KcFiuryKjXOB8lwlld2ypw.png)

First Step is to add a new data source which will be our prometheus

![](https://cdn-images-1.medium.com/max/2730/1*TjN6PXwfmuFPhg4uDX27EA.png)

then once the data source is connected i will use it to draw the first dashboard

The use case here is that i m going to monitor a multiple node-exporters so i will check in grafana-labs a template and here i find it
[**Node Exporter multiple Server Metrics | Grafana Labs**
*Dashboard to view multiple servers with node exporter version*grafana.com](https://grafana.com/grafana/dashboards/9990-node-exporter-server-metrics/)

so i will use this id to import it to my grafana

![](https://cdn-images-1.medium.com/max/2714/1*XYlJpvAi6iqozfdcylVsLw.png)

by clicking on load we load our dashboard and we can visualize the data

![](https://cdn-images-1.medium.com/max/2700/1*ORR3XsZnYnsBvUtHQo-M5Q.png)

and in the end to visualize some changes i just deployed a web application that i found on azure labs and watch the changes and it was not that remarquable change on the network and the i/o of disk

and by this here we are able to monitor our multi-cluster architecture by a simple cooperation between prometheus and grafana

Finally i hope that this article was useful and it was not that much long

feel free to contact me for any explanation and follow me for more content on my linkedin [https://www.linkedin.com/in/louay-k-77072083/](https://www.linkedin.com/in/louay-k-77072083/)
