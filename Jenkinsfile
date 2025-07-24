pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_Creds = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
    }

    options {
        disableResume()
        disableConcurrentBuilds(abortPrevious: true)
    }

    stages {
        stage('Installing Dependencies') {
            options {
                timestamps()
            }
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('NPM Dependency Audit') {
            steps {
                sh '''
                    npm audit --audit-level=critical
                    echo $?
                '''
            }
        }

        stage('Unit Testing') {
            options {
                retry(2)
            }
            steps {
                sh 'echo Colon-Separated - $MONGO_DB_Creds'
                sh 'echo Username - $MONGO_USERNAME'
                sh 'echo Password - $MONGO_PASSWORD'
                sh 'npm test'
            }
        }

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'Oops! It will be fixed in future releases', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'printenv'
                sh 'docker build -t poojadocker404/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'docker-hub-pat', variable: 'DOCKER_PAT')]) {
                    sh '''
                        echo "Logging in to Docker Hub..."
                        echo "$DOCKER_PAT" | docker login -u poojadocker404 --password-stdin

                        echo "Pushing image to Docker Hub..."
                        docker push poojadocker404/solar-system:$GIT_COMMIT
                    '''
                }
            }
        }

        stage('Deploy - AWS EC2') {
            when {
                 branch 'test'
            }
            steps {
                script {
                    sshagent(['aws-dev-deploy-ec2-instance']) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no ec2-user@ip-172-31-27-226 "
                            if sudo docker ps -a | grep -q 'solar-system'; then
                                echo "Container found. Stopping..."
                                sudo docker stop solar-system && sudo docker rm solar-system
                                echo "Container stopped and removed."
                            fi
                            sudo docker run --name solar-system \\
                                -e MONGO_URI=$MONGO_URI \\
                                -e MONGO_USERNAME=$MONGO_USERNAME \\
                                -e MONGO_PASSWORD=$MONGO_PASSWORD \\
                                -p 3000:3000 -d poojadocker404/solar-system:$GIT_COMMIT
                            "
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'test-results.xml'

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/lcov-report',
                reportFiles: 'index.html',
                reportName: 'Code Coverage HTML Report'
            ])
        }
    }
}
