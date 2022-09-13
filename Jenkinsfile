pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent{
                docker {
                    image 'openjdk:11'
                }
            }
            steps{
                  script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                         sh 'chmod +x gradlew'
                         sh './gradlew sonarqube'
                    }
                     timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }
               }
            }
        }
        stage("docker build & docker push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                             sh '''
                                docker build -t 172.16.18.131:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 172.16.18.131:8083 
                                docker push  172.16.18.131:8083/springapp:${VERSION}
                                docker rmi 172.16.18.131:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
         stage('identifying misconfigs using datree in helm charts'){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=ac0acd0a-f895-46f8-9879-9b2dd6eb3d80']) {
                              sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
         stage("pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                          dir('kubernetes/') {
                             sh '''
                                 helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                 tar -czvf  myapp-${helmversion}.tgz myapp/
                                 unset http_proxy
                                 unset https_proxy
                                 curl -u admin:$docker_password http://172.16.18.131:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                          }
                    }
                }
            }
        }
        stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }
        stage('Deploying application on k8s cluster') {
            steps {
               script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'Kubeconfig', namespace: '', serverUrl: 'https://172.16.18.128:6443') {
                        dir('kubernetes/') {
                          sh '''
                                unset http_proxy
                                unset https_proxy
                                helm upgrade --install --set image.repository="172.16.18.131:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ 
                             '''
                        }
                    }
               }
            }
        }
    }
}