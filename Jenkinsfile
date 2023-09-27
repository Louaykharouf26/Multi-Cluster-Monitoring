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