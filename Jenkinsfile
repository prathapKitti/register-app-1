pipeline {
    agent any
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_NAME = "kittappaprathapa/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/prathapKitti/register-app-1'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withCredentials([string(credentialsId: 'jenkins-sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarqube-server') {
                        sh '''
                            mvn sonar:sonar \
                            -Dsonar.projectKey=register-app \
                            -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    script {
                        docker.withRegistry('', 'dockerhub') {
                            def docker_image = docker.build("${IMAGE_NAME}")
                            docker_image.push("${IMAGE_TAG}")
                            docker_image.push("latest")
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                    --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table
                """
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_TOKEN')]) {
                    sh '''
                        curl -v -k --user admin:$JENKINS_TOKEN -X POST \
                        -H 'cache-control: no-cache' \
                        -H 'content-type: application/x-www-form-urlencoded' \
                        --data "IMAGE_TAG=$IMAGE_TAG" \
                        http://ec2-13-234-66-161.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token
                    '''
                }
            }
        }
    }

    // Optional Email Notification
    // post {
    //     failure {
    //         emailext body: '''${SCRIPT, template="groovy-html.template"}''',
    //                  subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Failed",
    //                  mimeType: 'text/html', to: "ashfaque.s510@gmail.com"
    //     }
    //     success {
    //         emailext body: '''${SCRIPT, template="groovy-html.template"}''',
    //                  subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Successful",
    //                  mimeType: 'text/html', to: "ashfaque.s510@gmail.com"
    //     }
    // }
}
