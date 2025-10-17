pipeline {
    agent {
        label 'ansible'
    }

    parameters {
        string(
            name: 'BACK_APP_VERSION',
            defaultValue: '',
            description: 'Backend version to deploy (leave empty to skip backend deployment)'
        )
        string(
            name: 'FRONT_APP_VERSION',
            defaultValue: '',
            description: 'Frontend version to deploy (leave empty to skip frontend deployment)'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['DEV', 'PROD'],
            description: 'Target environment for deployment'
        )
    }

    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        NEXUS_USERNAME = "${env.NEXUS_CREDENTIALS_USR}"
        NEXUS_PASSWORD = "${env.NEXUS_CREDENTIALS_PSW}"
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    echo "=== Deployment Configuration ==="
                    echo "Target Environment: ${params.TARGET_ENV}"
                    echo "Backend Version: ${params.BACK_APP_VERSION ?: 'SKIP'}"
                    echo "Frontend Version: ${params.FRONT_APP_VERSION ?: 'SKIP'}"
                    echo "================================"

                    if (!params.BACK_APP_VERSION && !params.FRONT_APP_VERSION) {
                        error("At least one version (BACK_APP_VERSION or FRONT_APP_VERSION) must be specified")
                    }

                    // Set deployment flags
                    env.DEPLOY_BACKEND = params.BACK_APP_VERSION ? 'true' : 'false'
                    env.DEPLOY_FRONTEND = params.FRONT_APP_VERSION ? 'true' : 'false'
                }
            }
        }

        stage('Prepare Environment') {
            steps {
                script {
                    // Get SSH password for magnolia host
                    withCredentials([string(credentialsId: 'magnolia-ssh-password', variable: 'SSH_PASS')]) {
                        env.SSH_PASSWORD = SSH_PASS
                    }
                    // Get HTTP auth password for version verification
                    withCredentials([string(credentialsId: 'webuser-http-password', variable: 'HTTP_PASS')]) {
                        env.HTTP_AUTH_PASSWORD = HTTP_PASS
                    }
                }
            }
        }

        stage('Deploy Backend') {
            when {
                expression { env.DEPLOY_BACKEND == 'true' }
            }
            steps {
                script {
                    echo "=== Deploying Backend Version ${params.BACK_APP_VERSION} to ${params.TARGET_ENV} ==="

                    sh """
                        cd ansible
                        ansible-playbook -i inventory.ini deploy.yml \
                            -e "target_env=${params.TARGET_ENV}" \
                            -e "app_type=back" \
                            -e "back_app_version=${params.BACK_APP_VERSION}" \
                            -v
                    """

                    echo "Backend deployment completed successfully"
                }
            }
        }

        stage('Deploy Frontend') {
            when {
                expression { env.DEPLOY_FRONTEND == 'true' }
            }
            steps {
                script {
                    echo "=== Deploying Frontend Version ${params.FRONT_APP_VERSION} to ${params.TARGET_ENV} ==="

                    sh """
                        cd ansible
                        ansible-playbook -i inventory.ini deploy.yml \
                            -e "target_env=${params.TARGET_ENV}" \
                            -e "app_type=front" \
                            -e "front_app_version=${params.FRONT_APP_VERSION}" \
                            -v
                    """

                    echo "Frontend deployment completed successfully"
                }
            }
        }
    }

    post {
        always {
            echo '=== Deployment Pipeline Completed ==='
        }
        success {
            script {
                def deployedApps = []
                if (env.DEPLOY_BACKEND == 'true') {
                    deployedApps.add("Backend ${params.BACK_APP_VERSION}")
                }
                if (env.DEPLOY_FRONTEND == 'true') {
                    deployedApps.add("Frontend ${params.FRONT_APP_VERSION}")
                }
                echo "SUCCESS: ${deployedApps.join(' and ')} deployed to ${params.TARGET_ENV}"
            }
        }
        failure {
            echo 'FAILURE: Deployment failed. Check logs above for details.'
        }
    }
}
