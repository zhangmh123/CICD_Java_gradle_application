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
    }
}