pipeline {
    agent any
    tools {
        maven "MAVEN3.9.9"
        jdk "JDK21"
    }  

    environment {        
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        IMAGE_NAME = 'ndzenyuy/ecommerce-app'
        IMAGE_TAG  = 'latest'
        TASK_DEF_ARN = 'arn:aws:ecs:us-east-1:997450571655:task-definition/jenkins-cicd-task'
        SONAR_PROJECTKEY= 'jenkins-cicd1_project'
        SONAR_PROJECTNAME= 'project'
        SONAR_ORG= 'jenkins-cicd1'
        ECS_CLUSTER = 'jenkins-cicd-cluster'
        ECS_SERVICE = 'jenkins-cicd-service'
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

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
               withSonarQubeEnv("${SONARSERVER}") {
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey="$SONAR_PROJECTKEY" \
                   -Dsonar.projectName="$SONAR_PROJECTNAME" \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.organization="$SONAR_ORG" \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
              }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                sh '''
                echo "Installing unzip..."
                sudo apt-get update && sudo apt-get install -y unzip

                echo "Downloading OWASP Dependency-Check CLI..."
                curl -L -o dependency-check.zip https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip

                echo "Unzipping..."
                unzip -o -q dependency-check.zip -d dependency-check-dir

                echo "Listing dependency-check-dir contents:"
                ls -l dependency-check-dir/dependency-check/bin/

                echo "Setting executable permission..."
                chmod +x dependency-check-dir/dependency-check/bin/dependency-check.sh

                echo "Running OWASP Dependency-Check..."
                ./dependency-check-dir/dependency-check/bin/dependency-check.sh --version || echo "Failed to run dependency-check.sh"

                ./dependency-check-dir/dependency-check/bin/dependency-check.sh \
                    --project "MyProject" \
                    --scan . \
                    --format HTML \
                    --out owasp-report \
                    --failOnCVSS 7 || true

                echo "OWASP scan complete, reports in owasp-report/"
                '''
            }
        }

        stage('Building image') {
            steps{
              script {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
                sh 'docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:$IMAGE_TAG'
              }
            }
        } 

        stage('Trivy Scan') {
                steps {
                    script {
                        sh 'trivy image --severity HIGH,CRITICAL --format table $IMAGE_NAME:$BUILD_NUMBER' // Scan for high/critical vulnerabilities
                        // You can also output to a file:
                         sh 'trivy image $IMAGE_NAME:$BUILD_NUMBER > trivy-report.txt'
                    }
                }
         } 
        stage('Push to Dockerhub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKER_LOGIN',  // ID from Jenkins credentials
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]){
                    sh'''                    
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push $IMAGE_NAME:$BUILD_NUMBER
                      docker push $IMAGE_NAME:$IMAGE_TAG
                      docker logout                      
                    '''                   
                }
                
            }
        } 
        stage('Remove Images')   {
            steps {
                script {
                    sh 'docker rmi  $IMAGE_NAME:$BUILD_NUMBER'
                    sh 'docker rmi  $IMAGE_NAME:$IMAGE_TAG'
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
                channel: '#jenkinscicd',
                color: currentBuild.currentResult == 'SUCCESS' ? 'good' : 'danger',
                message: "The recently built Pipeline *${env.JOB_NAME}* #${env.BUILD_NUMBER} finished with status: *${currentBuild.currentResult}*\n${env.BUILD_URL}"
            )
        }

        success {
            emailext(
                to: 'ndzenyuyjones@gmail.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' succeeded.</p>
                        <p>Check console output at <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>""",
                mimeType: 'text/html'
            )
        }

        failure {
            emailext(
                to: 'ndzenyuyjones@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed.</p>
                        <p>Check console output at <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>""",
                mimeType: 'text/html'

            )
        }
    }
}