pipeline {
    agent any

    options {
        skipDefaultCheckout() // Default repository cloning fails, causing pipeline failure
    }

    stages {
        stage('Get Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Alfonso-Caballero/todo-list-aws.git'
                
                script {
                        def fileUrl = 'https://github.com/Alfonso-Caballero/todo-list-aws-config/raw/production/samconfig.toml'
                        def fileName = 'samconfig.toml'
                    
                        bat "curl -o %WORKSPACE%\\${fileName} ${fileUrl}"
                    
                        bat "type %WORKSPACE%\\${fileName}"
                        }
                
                bat 'whoami'
                bat 'hostname'
                echo "${WORKSPACE}"
                
                stash name: 'code', includes: '**'
                }
            post {
                always {
                        deleteDir()
                        }
                    } 
        }
        stage('Deploy') {
            agent {
                label 'ec2'
            }
            steps {
                unstash 'code'
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
                                    --stack-name my-production-stack \
                                    --capabilities CAPABILITY_IAM \
                                    --no-confirm-changeset \
                                    --region us-east-1 \
                                    --s3-bucket bucketnugget \
                                    --parameter-overrides \
                                        Stage=production \
                                        Environment=production \
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
                            def pytestResult = sh(script: "pytest -m read_only --junitxml=result-rest.xml test/integration/todoApiTest.py", returnStatus: true)
                            echo "Pytest returned status: ${pytestResult}"
                            if (pytestResult == 0) {
                                echo "Pytest executed successfully."
                                junit 'result*.xml'
                            } else if (pytestResult == 5) {  // Código de salida 5 indica que no se encontraron pruebas
                                echo "No read-only tests found. Skipping pytest execution."
                            } else {
                                currentBuild.result = 'FAILURE'
                                error "Test execution failed with status: ${pytestResult}"
                            }
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
        }
}
