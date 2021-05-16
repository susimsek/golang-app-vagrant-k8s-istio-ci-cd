/*
    Create the kubernetes namespace
 */
def createNamespace (namespace) {
    echo "Creating namespace ${namespace} if needed"

    sh "[ ! -z \"\$(kubectl get ns ${namespace} -o name 2>/dev/null)\" ] || kubectl create ns ${namespace}"
}

/*
    Helm install
 */
def helmInstall (namespace, release) {
    echo "Installing ${release} in ${namespace}"

    script {
        sh """
            helm upgrade --install --namespace ${namespace} ${release} ${CHART_DIR}
        """
    }
}

pipeline {
    environment {
        BRANCH_NAME = "${env.GIT_BRANCH.split("/")[1]}"
        DEPLOY = "${BRANCH_NAME == "main" || BRANCH_NAME == "develop" ? "true" : "false"}"
        CHART_NAME = "app"
        CHART_NAMESPACE= "app"
        CHART_DIR = "helm-chart/app"
        IMAGE_REGISTRY = 'suayb/golang-echo-graphql-example'
        IMAGE_VERSION = '1.0.0'
        IMAGE_REGISTRY_CREDENTIAL = 'dockerhubcreds'
    }
    agent {
        kubernetes {
            defaultContainer 'jnlp'
            yamlFile 'build.yaml'
        }
    }
    stages {
        stage('Kubernetes Version Control') {
             when {
                environment name: 'DEPLOY', value: 'true'
             }
             steps {
                container('kubectl') {
                    sh "kubectl version"
                }
             }
        }
        stage('Build') {
             when {
                 environment name: 'DEPLOY', value: 'true'
             }
            steps {
               container('golang') {
                   sh """
                      ln -s `pwd` /go/src/app
                      cd /go/src/app
                      go mod download
                      go build -v -o server
                       """
               }
            }
        }
        stage('Docker Build') {
             when {
                environment name: 'DEPLOY', value: 'true'
             }
            steps {
                container('docker') {
                    sh "docker build -t ${IMAGE_REGISTRY}:${IMAGE_VERSION} ."
                }
            }
        }
        stage('Docker Publish') {
            when {
                environment name: 'DEPLOY', value: 'true'
            }
            steps {
                container('docker') {
                    withDockerRegistry([credentialsId: "${IMAGE_REGISTRY_CREDENTIAL}", url: ""]) {
                        sh "docker push ${IMAGE_REGISTRY}:${IMAGE_VERSION}"
                    }
                }
            }
        }
        stage('Kubernetes Deploy') {
              when {
                    environment name: 'DEPLOY', value: 'true'
              }
              steps {
                script {
                  container('kubectl') {
                        createNamespace (CHART_NAMESPACE)
                  }
                  container('helm') {
                        helmInstall(CHART_NAMESPACE, CHART_NAME)
                  }
                }
              }
        }
    }
}