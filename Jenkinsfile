pipeline {
    agent any

    options {
        skipDefaultCheckout() // Default repository cloning fails, causing pipeline failure
    }
    
    environment {
        GIT_TOKEN = credentials('token')
    }

    stages {
        stage('Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/Alfonso-Caballero/todo-list-aws.git'
                bat 'whoami'
                bat 'hostname'
                echo "${WORKSPACE}"
                stash name: 'code', includes : '**'
                
            }   
        }
        stage('Static Test') {
            agent {
                label 'ec2'
            }
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                            unstash 'code'
                            sh '''
                                (bandit -r ./src -f custom -o bandit.out -l --msg-template "{abspath}:{line}: [{test_id}] {msg}") || exit 0 
                                '''
                            recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], qualityGates: [[threshold: 2, type: 'TOTAL', unstable: true], [threshold: 4, type: 'TOTAL', unstable: false]]
                            sh '''
                                python3 -m flake8 --exit-zero --format=pylint src >flake8.out
                                '''
                            recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates: [[threshold: 8, type: 'TOTAL', unstable: true], [threshold: 10, type: 'TOTAL', unstable: false]]
                            sh 'whoami'
                            sh 'hostname'
                            echo "${WORKSPACE}"
                        }
                    }
        }
        stage('Deploy') {
            agent {
                label 'ec2'
            }
            steps {
                sh 'whoami'
                sh 'hostname'
                echo "${WORKSPACE}"
                sh 'sam build'
                sh 'sam validate --region us-east-1'
                script{
                    def deployOutput = sh(
                                script: '''
                                    sam deploy \
                                    --template-file .aws-sam/build/template.yaml \
                                    --stack-name my-staging-stack \
                                    --capabilities CAPABILITY_IAM \
                                    --no-confirm-changeset \
                                    --region us-east-1 \
                                    --s3-bucket bucketnugget \
                                    --parameter-overrides \
                                        Stage=staging \
                                        Environment=staging \
                                        ParameterKey1=Value1 \
                                        ParameterKey2=Value2 \
                                    --force-upload
                                ''',
                                returnStdout: true
                            ).trim()
                            
                            // Extraer la URL de salida (BaseUrlApi)
                            def baseUrlMatch = deployOutput =~ /Value\s+(.+)/
                            def baseUrl = baseUrlMatch[0][1].trim()
                            
                            echo "Deployed successfully. Base URL: ${baseUrl}"
                            
                            // Ejecutar las pruebas de integración con la URL capturada como parámetro
                           env.BASE_URL = baseUrl
                    }
            }
        }
        stage('Rest Test') {
            agent {
                label 'ec2'
            }
            steps {
                script {
                    sh 'whoami'
                    sh 'hostname'
                    echo "${WORKSPACE}"
                    try {
                        sh "pytest --junitxml=result-rest.xml test/integration/todoApiTest.py"
                        junit 'result*.xml'
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        error "Test execution failed: ${e.message}"
                    }
                }
            }
            post {
                always {
                    script {
                        if (currentBuild.result == 'FAILURE') {
                            error "Pipeline failed. Check logs for details."
                        }      
                    }
                    deleteDir()
                }
            }
        }
        
        stage('Promote') {
            steps {
                withCredentials([string(credentialsId: 'token', variable: 'GIT_TOKEN')]) {
                    bat """
                    @echo off
                    setlocal

                    REM Clone the main branch using the token
                    git clone -b main https://%GIT_TOKEN%@github.com/Alfonso-Caballero/todo-list-aws.git main-repo
                    cd main-repo

                    REM Add the develop branch as a remote and fetch it
                    git remote add develop https://%GIT_TOKEN%@github.com/Alfonso-Caballero/todo-list-aws.git
                    git fetch develop develop

                    REM Merge the develop branch into main using the token
                    git merge develop/develop --no-ff --no-edit

                    REM Push the changes to the main branch using the token
                    git push origin main

                    endlocal
                    """
                }
                bat 'whoami'
                bat 'hostname'
                echo "${WORKSPACE}"
                echo "Code successfully merged into main."
                    
            }
            post {
                    always {
                        deleteDir()
                        }
                    }   
                }
        }
}
