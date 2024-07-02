pipeline {
    agent any

    environment {
        GIT_REPO_URL = 'https://github.com/Alfonso-Caballero/todo-list-aws.git'
        GIT_BRANCH = 'develop'
        GIT_CREDENTIALS_ID = 'git_token'
        PATH = "/home/ubuntu/.local/bin:/usr/local/bin:/usr/bin:/bin"
        PYTHONPATH = "/var/lib/jenkins/workspace/retoD_1"
    }

    stages {
        stage('Get Code') {
            steps {
                script {
                    // Utiliza las credenciales para clonar el repositorio
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
        stage('Static') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                            sh 'whoami'
                            sh 'hostname'
                            echo "${WORKSPACE}"
                            
                            sh 'bandit --version'
                        }
                    }
            /*
                    post {
                        always {
                            deleteDir()
                }
            }
            */
        // Puedes añadir más etapas aquí según sea necesario
        }
    }
}
