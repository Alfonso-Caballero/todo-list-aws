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
                    bat 'whoami'
                    bat 'hostname'
                    echo "${WORKSPACE}"
                    stash name: 'code', includes : '**'
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
        // Puedes añadir más etapas aquí según sea necesario
        }
    }
}
