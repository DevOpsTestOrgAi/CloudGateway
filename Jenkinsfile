pipeline {
    agent any

    environment {
        registryName = 'sk09devops/cloud-gateway' // Updated Docker Hub repository name
        registryCredential = 'DOCKERHUB' // Updated credential name
        dockerImage = ''
        registryUrl = ''
        mvnHome = tool name: 'maven', type: 'maven'
        mvnCMD = "${mvnHome}/bin/mvn "
        imageTag = "latest-${BUILD_NUMBER}" // Default tag with build number
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    deleteDir()
                    checkout([$class: 'GitSCM',
                              branches: [[name: 'main']],
                              doGenerateSubmoduleConfigurations: false,
                              extensions: [[$class: 'CleanBeforeCheckout'], [$class: 'RelativeTargetDirectory', relativeTargetDir: 'Api']],
                              submoduleCfg: [],
                              userRemoteConfigs: [[url: 'https://github.com/DevOpsTestOrgAi/CloudGateway.git']]])
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    dir('Api') {
                        sh "${mvnCMD} clean install"
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('Sonar') {
                        dir('Api'){
                            sh "${mvnCMD} sonar:sonar"
                        }
                    }
                    slackSend message: 'AI-Extension ai extension: Sonar analysis completed. Check http://172.174.23.176:9000/'
                }
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                    dir('Api') {
                        // Append the build number to the "latest" tag
                        imageTag = "latest-${BUILD_NUMBER}"
                        dockerImage = docker.build(registryName, "-f Dockerfile . --tag ${imageTag}")
                    }
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry("https://registry.hub.docker.com", registryCredential) {
                        dockerImage.push("${imageTag}")
                    }
                }
                slackSend message: "AI-Extension cloud gateway: New Artifact was Pushed to Docker Hub Repo with tag ${imageTag}"
            }
        }
        stage('Update Manifests and Push to Git') {
            steps {
                script {
                  
                    def cloneDir = 'GitOps'

                  
                    if (!fileExists(cloneDir)) {
                        sh "git clone https://github.com/DevOpsTestOrgAi/GitOps.git ${cloneDir}"
                    }

                   
                    def manifestsDir = "${cloneDir}/k8s"

                  
                    
                    def newImageLine = "image: ${registryName}:${imageTag}"
            
                    sh "sed -i 's|image: sk09devops/ai-project:latest.*|${newImageLine}|' ${manifestsDir}/cloudgateway-deployment.yml"


                    
                    withCredentials([usernamePassword(credentialsId: 'git',passwordVariable: 'GIT_PASSWORD' , usernameVariable: 'GIT_USERNAME')]) {
                        dir(cloneDir) {
                            sh "git config user.email mohamedammaha2020@gmail.com"
                            sh "git config user.name medXPS"
                            sh "git add ."
                            sh "git commit -m 'Update image tag in Kubernetes manifests'"
                            sh "git push  https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/DevOpsTestOrgAi/GitOps.git HEAD:main"
                        }
                    }
                }
            }
        }

    }
}

