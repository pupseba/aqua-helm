@Library('aqua-pipeline-lib@master')_

def charts = [ 'server', 'kube-enforcer', 'enforcer', 'gateway', 'aqua-quickstart', 'cyber-center', 'cloud-connector' ]
pipeline {
    agent {
        label 'automation_slaves'
    }
    /*
    environment {
        AQUASEC_AZURE_ACR_PASSWORD = credentials('aquasecAzureACRpassword')
        AFW_SERVER_LICENSE_TOKEN = credentials('AquaDeploymentLicenseToken')
        ROOT_CA = credentials('deployment_ke_webook_root_ca')
        SERVER_CERT = credentials('deployment_ke_webook_crt')
        SERVER_KEY = credentials('deployment_ke_webook_key')
    } */
    options {
        ansiColor('xterm')
        timestamps()
        skipStagesAfterUnstable()
        skipDefaultCheckout()
        buildDiscarder(logRotator(daysToKeepStr: '7'))
    }
    stages {
        stage('Checkout') {
            steps {
                checkout([
                        $class: 'GitSCM',
                        branches: scm.branches,
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [],
                        submoduleCfg: [],
                        userRemoteConfigs: scm.userRemoteConfigs
                ])              
            }
        }
        stage("Lint Checking") {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    reuseNode true
                    }
            }
            steps {
                script {
                    for ( int i=0; i < charts.size(); i++) {
                        sh "helm lint ${charts[i]}"
                    }
                }
            }
        }
        stage("Kubeval Checkings") {
                agent {
                    dockerfile {
                        filename 'Dockerfile'
                        reuseNode true
                        }
                }
                steps {
                    script {
                        sh "helm dependency update server/"
                        for ( int i=0; i < charts.size(); i++) {
                            sh "helm template ${charts[i]}/ --set global.platform=k8s,platform=k8s,imageCredentials.username=test,imageCredentials.password=test,webhooks.caBundle=test,certsSecret.serverCertificate=test,certsSecret.serverKey=test,user=test,password=test > ${charts[i]}.yaml && \
                            kubeval ${charts[i]}.yaml --ignore-missing-schemas"
                        }
                    }
                }
        }
        /*
        stage("Creating K3s Cluster") {
            steps {
                sh 'curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -'
                echo 'k3s installed'
            }
        }
        stage("Deploying Aqua Charts") {
            steps {
                sh '''
                    wget -q https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz
                    tar -zxvf helm-v3.7.2-linux-amd64.tar.gz
                    mkdir -pv local/bin
                    mv linux-amd64/helm local/bin/
                '''
                sh 'cp /etc/rancher/k3s/k3s.yaml ~/.kube/config'
                sh 'export KUBECONFIG=~/.kube/config'
                sh 'kubectl get nodes -o wide && kubectl create namespace aqua'
                script {
                    log.info "installing server chart"
                    def TOKEN = sh script: "az acr login --name 'aquasec' --expose-token -o json", returnStdout: true
                    def exposedTokenJSON = readJSON text: "${TOKEN}"
                    log.info "creating aqua namespace"
                    sh "kubectl create secret -n aqua docker-registry aquasec-registry --docker-server=aquasec.azurecr.io  --docker-username=00000000-0000-0000-0000-000000000000  --docker-password=\\${exposedTokenJSON['accessToken']}\n"
                    log.info "installing server chart"
                    sh('local/bin/helm upgrade --install --namespace aqua server server/ --set global.platform=k3s,gateway.service.type=LoadBalancer,imageCredentials.create=false,imageCredentials.name=aquasec-registry,imageCredentials.repositoryUriPrefix=aquasec.azurecr.io,gateway.imageCredentials.repositoryUriPrefix=aquasec.azurecr.io,admin.password=HelmAquaCI@123,admin.token=$AFW_SERVER_LICENSE_TOKEN')
                    log.info "Installing enforcer chart"
                    sh('local/bin/helm upgrade --install --namespace aqua enforcer enforcer/ --set imageCredentials.create=false,imageCredentials.name=aquasec-registry,imageCredentials.repositoryUriPrefix=aquasec.azurecr.io,platform=k3s')
                    log.info "Installing Kube-enforcer chart"
                    sh('local/bin/helm upgrade --install --namespace aqua kube-enforcer/ --set imageCredentials.create=false,imageCredentials.name=aquasec-registry,imageCredentials.repositoryUriPrefix=aquasec.azurecr.io,platform=k3s,webhooks.caBundle=$ROOT_CA,certsSecret.serverCertificate=$SERVER_CERT,certsSecret.serverKey=$SERVER_KEY')
                }
                sleep(120)
                sh "kubectl get pods -n aqua && kubectl get svc -n aqua"
            }
        }
        stage("Validating") {
            parallel {
                stage("pods state") {
                    steps {
                        script {
                            log.info "checking all pods are running or not"
                            def bs = """kubectl get pods -n aqua  | awk '{print \$3}' |grep -v STATUS | grep -v Running"""
                            def status = sh returnStatus:true ,script: bs
                            if (status == 0) {
                                echo "${status}"
                                log.warning "Found issues in aqua namespace"
                                script.sh("kubectl describe pods -n aqua >> describe_pods.log")
                                archiveArtifacts artifacts: 'describe_pods.log', onlyIfSuccessful: true
                            }
                            else if (status == 1) {
                                echo "${status}"
                                log.info "all pods are running"
                            }
                        }
                    }
                }
                stage("Server endpoint") {
                    steps {
                        sh "kubectl get pods -n aqua"
                        script {
                            sh "kubectl get svc -n aqua"
                            def STATUS = sh (script: "curl -o /dev/null --connect-timeout 5 -s -w '%{http_code}' localhost:8080", returnStdout: true)
                            if (STATUS == '200') {
                                log.info("server up and running")
                            } else {
                                log.warn("Issue with server connectivity")
                            }
                        }
                    }
                }
            }
        }
        */
        stage("Pushing Helm chart to dev repo") {
            agent {
                docker {
                    image 'alpine:latest'
                    args '-u root'
                    reuseNode true
                    }
            }
            steps {
                script {
                    sh 'apk add --no-cache ca-certificates git tar && tar -zxvf helm-v3.7.2-linux-amd64.tar.gz && mv linux-amd64/helm /usr/local/bin'
                    sh 'helm plugin install https://github.com/chartmuseum/helm-push.git'
                    sh 'helm plugin list'
                    sh 'helm repo add aqua-dev https://helm-dev.aquaseclabs.com/'
                    sh 'helm repo list'
                    sh 'helm cm-push server/  aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push tenant-manager/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push enforcer/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push gateway/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push aqua-quickstart/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push kube-enforcer/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push cyber-center/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                    sh 'helm cm-push cloud-connector/ aqua-dev --version="6.5-${JOB_NAME##*/}-${BUILD_NUMBER}"'
                }
            }
        }
        }
    post {
        always {
            script {
                /* 
                sh "local/bin/helm uninstall server enforcer -n aqua"
                sh "sh /usr/local/bin/k3s-uninstall.sh"
                echo "k3s & server chart uninstalled"
                */
                cleanWs()
//                notifyFullJobDetailes subject: "${env.JOB_NAME} Pipeline | ${currentBuild.result}", emails: userEmail
            }
        }
    }
}
