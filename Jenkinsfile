pipeline {
    agent any
    tools {
        maven "MAVEN3.9.9"
        jdk "JDK21"
    }  

    environment {        
        SONARSERVER      = 'sonarserver' //sonarserver
        IMAGE_NAME       = 'nelvis1/cloudreality-image'
        IMAGE_TAG        = 'latest' 
        TASK_DEF_ARN     = 'arn:aws:ecs:us-east-1:997450571655:task-definition/jenkins-cicd-task'
        SONAR_PROJECTKEY = 'jenkinscicdproject' 
        SONAR_PROJECTNAME= 'jenkinscicdproject'
        SONAR_ORG        = 'jenkinscicd'
        ECS_CLUSTER      = 'jenkins-cicd-cluster'
        ECS_SERVICE      = 'jenkins-cicd-service'
    }
 
    stages {
        stage('Build'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test'){
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarserver') {
                        sh 'mvn verify sonar:sonar \
                            -Dsonar.projectKey=$SONAR_PROJECTKEY \
                            -Dsonar.organization=$SONAR_ORG \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=$SONAR_TOKEN'
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh '''
                sudo apt-get update && sudo apt-get install -y unzip
                curl -L -o dependency-check.zip https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip
                unzip -o -q dependency-check.zip -d dependency-check-dir
                chmod +x dependency-check-dir/dependency-check/bin/dependency-check.sh
                ./dependency-check-dir/dependency-check/bin/dependency-check.sh \
                    --project "MyProject" \
                    --scan . \
                    --format HTML \
                    --out owasp-report \
                    --failOnCVSS 7 || true
                '''
            }
        }

        stage('Building Image') {
            steps {
                script {
                    sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
                    sh 'docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:$IMAGE_TAG'
                }
            }
        } 

        stage('Trivy Scan') {
            steps {
                script {
                    sh 'trivy image --severity HIGH,CRITICAL --format table $IMAGE_NAME:$BUILD_NUMBER'
                    sh 'trivy image $IMAGE_NAME:$BUILD_NUMBER > trivy-report.txt'
                }
            }
        } 

        stage('Push to Dockerhub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKER_LOGIN',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''                    
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        docker logout                      
                    '''                   
                }
            }
        } 

        stage('Remove Images') {
            steps {
                script {
                    sh 'docker rmi $IMAGE_NAME:$BUILD_NUMBER'
                    sh 'docker rmi $IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }  

        stage('Update ECS Task Definition') {
            steps {
                script {
                    sh '''aws ecs describe-task-definition --task-definition "$TASK_DEF_ARN" --query 'taskDefinition.{family: family, taskRoleArn: taskRoleArn, executionRoleArn: executionRoleArn, networkMode: networkMode, containerDefinitions: containerDefinitions, volumes: volumes, placementConstraints: placementConstraints, requiresCompatibilities: requiresCompatibilities, cpu: cpu, memory: memory}' --output json > task-def.json'''
                    
                    def taskDefinition = readFile('task-def.json')
                    def newTaskDefinition = taskDefinition.replaceAll(/"image":\\s*".*?"/, '"image": "' + IMAGE_NAME + ':' + IMAGE_TAG + '"')
                    writeFile file: 'new-task-definition.json', text: newTaskDefinition                 

                    sh 'aws ecs register-task-definition --cli-input-json file://new-task-definition.json'
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    sh 'aws ecs update-service --cluster "$ECS_CLUSTER" --service "$ECS_SERVICE" --force-new-deployment'
                }
            }            
        } 
    }
          
    post {
        always {
            slackSend(
                channel: '#all-cloudreality',
                color: currentBuild.currentResult == 'SUCCESS' ? 'good' : 'danger',
                message: "The recently built Pipeline *${env.JOB_NAME}* #${env.BUILD_NUMBER} finished with status: *${currentBuild.currentResult}*\n${env.BUILD_URL}"
            )
        }

        success {
            emailext(
                to: 'ntsezevouffonelvis@gmail.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' succeeded.</p>
                         <p>Check console output at <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>""",
                mimeType: 'text/html'
            )
        }

        failure {
            emailext(
                to: 'ntsezevouffonelvis@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed.</p>
                         <p>Check console output at <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>""",
                mimeType: 'text/html'
            )
        }
    }
}
