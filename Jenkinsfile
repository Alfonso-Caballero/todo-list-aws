pipeline {
    agent any

    options {
        skipDefaultCheckout() // Default repository cloning fails, causing pipeline failure
    }

    environment {
        GIT_REPO_URL = 'https://github.com/Alfonso-Caballero/todo-list-aws.git'
        GIT_BRANCH = 'develop'
        GIT_CREDENTIALS_ID = 'git_token'
    }

    stages {
        stage('Get Code') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM', 
                        branches: [[name: "refs/heads/${env.GIT_BRANCH}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [],
                        userRemoteConfigs: [[
                            url: env.GIT_REPO_URL,
                            credentialsId: env.GIT_CREDENTIALS_ID
                        ]]
                    ])
                    bat 'whoami'
                    bat 'hostname'
                    echo "${WORKSPACE}"
                    stash name: 'code', includes : '**'
                }
            }
            
            post {
                    always {
                        deleteDir()
                        }
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
            /*
                    post {
                        always {
                            deleteDir()
                }
            }
            */
        }
        stage('Deploy') {
            agent {
                label 'ec2'
            }
            steps {
                sh "aws cloudformation delete-stack --stack-name my-staging-stack"
                echo "Waiting for stack deletion..."
                sh "aws cloudformation wait stack-delete-complete --stack-name my-staging-stack"
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
                /*
                sh '''
                        sam deploy \
                        --template-file .aws-sam/build/template.yaml \
                        --stack-name my-staging-stack \
                        --capabilities CAPABILITY_IAM \
                        --no-confirm-changeset \
                        --region us-east-1 \
                        --s3-bucket bucketnugget \
                        --parameter-overrides \
                            Environment=staging \
                            ParameterKey1=Value1 \
                            ParameterKey2=Value2 \
                        --force-upload
                    '''
                    */
            }
        }
        stage('Rest Test') {
            agent {
                label 'ec2'
            }
            steps {
                script {
                    try {
                        sh "pytest test/integration/todoApiTest.py"
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
            }
            deleteDir()
        }
        post {
            always {
                    
                    }
                }
    }
        /*
        stage('Rest Test') {
            agent {
                label 'ec2'
            }
            steps {
                script {
                    sh "pytest test/integration/todoApiTest.py"
                }
            }
            */
            
            
                    
    }
}
