pipeline {
    agent {
        label 'ansible'
    }

    parameters {
        choice(
            name: 'TARGET_ENV',
            choices: ['DEV', 'PROD', 'DOCKER_COMPOSE'],
            description: 'Target environment: DEV/PROD (Ansible VM deployment) or DOCKER_COMPOSE (local container deployment)'
        )
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
    }

    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    environment {
        ANSIBLE_FORCE_COLOR = 'true'
        PY_COLORS = '1'
        ANSIBLE_NOCOLOR = '0'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        MYSQL_CREDENTIALS = credentials('mysql-credentials')
        JWT_SECRET = credentials('jwt-secret')
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    echo "=== Deployment Configuration ==="
                    echo "Target Environment: ${params.TARGET_ENV}"
                    echo "Backend Version: ${params.BACK_APP_VERSION ?: 'latest'}"
                    echo "Frontend Version: ${params.FRONT_APP_VERSION ?: 'latest'}"
                    echo "================================"

                    // Versions required for Ansible deployments, optional for Docker Compose (defaults to latest)
                    if (params.TARGET_ENV in ['DEV', 'PROD'] && !params.BACK_APP_VERSION && !params.FRONT_APP_VERSION) {
                        error("At least one version (BACK_APP_VERSION or FRONT_APP_VERSION) must be specified for ${params.TARGET_ENV} deployment")
                    }

                    // Set deployment flags
                    env.DEPLOY_BACKEND = params.BACK_APP_VERSION ? 'true' : 'false'
                    env.DEPLOY_FRONTEND = params.FRONT_APP_VERSION ? 'true' : 'false'

                    // Set build display name with versions
                    def buildName = "${params.TARGET_ENV}:"
                    if (params.TARGET_ENV == 'DOCKER_COMPOSE') {
                        buildName += " Back ${params.BACK_APP_VERSION ?: 'latest'}"
                        buildName += " Front ${params.FRONT_APP_VERSION ?: 'latest'}"
                    } else {
                        if (params.BACK_APP_VERSION) {
                            buildName += " Back ${params.BACK_APP_VERSION}"
                        }
                        if (params.FRONT_APP_VERSION) {
                            buildName += " Front ${params.FRONT_APP_VERSION}"
                        }
                    }
                    currentBuild.displayName = "#${env.BUILD_NUMBER} - ${buildName}"
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
                allOf {
                    expression { params.TARGET_ENV in ['DEV', 'PROD'] }
                    expression { env.DEPLOY_BACKEND == 'true' }
                }
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
                allOf {
                    expression { params.TARGET_ENV in ['DEV', 'PROD'] }
                    expression { env.DEPLOY_FRONTEND == 'true' }
                }
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

        stage('Deploy Docker Compose') {
            agent {
                label 'docker'
            }
            when {
                expression { params.TARGET_ENV == 'DOCKER_COMPOSE' }
            }
            steps {
                script {
                    echo "=== Deploying with Docker Compose ==="

                    // Set ports and profile for local Docker deployment
                    // Using non-default ports to avoid conflicts with ng serve (4200), Spring Boot (8080/8081), and MySQL (3306)
                    def backPort = '9081'
                    def frontPort = '3000'
                    def mysqlPort = '3307'
                    def springProfile = 'dev'

                    // Set versions (use latest if not specified)
                    def backVersion = params.BACK_APP_VERSION ?: 'latest'
                    def frontVersion = params.FRONT_APP_VERSION ?: 'latest'

                    // Login to DockerHub to pull images
                    sh """
                        echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                    """

                    // Deploy with docker-compose
                    sh """
                        cd docker
                        BACK_VERSION=${backVersion} \\
                        FRONT_VERSION=${frontVersion} \\
                        BACK_PORT=${backPort} \\
                        FRONT_PORT=${frontPort} \\
                        MYSQL_PORT=${mysqlPort} \\
                        SPRING_PROFILE=${springProfile} \\
                        MYSQL_USER=${MYSQL_CREDENTIALS_USR} \\
                        MYSQL_PASSWORD=${MYSQL_CREDENTIALS_PSW} \\
                        MYSQL_ROOT_PASSWORD=${MYSQL_CREDENTIALS_PSW} \\
                        JWT_SECRET=${JWT_SECRET} \\
                        docker-compose pull

                        BACK_VERSION=${backVersion} \\
                        FRONT_VERSION=${frontVersion} \\
                        BACK_PORT=${backPort} \\
                        FRONT_PORT=${frontPort} \\
                        MYSQL_PORT=${mysqlPort} \\
                        SPRING_PROFILE=${springProfile} \\
                        MYSQL_USER=${MYSQL_CREDENTIALS_USR} \\
                        MYSQL_PASSWORD=${MYSQL_CREDENTIALS_PSW} \\
                        MYSQL_ROOT_PASSWORD=${MYSQL_CREDENTIALS_PSW} \\
                        JWT_SECRET=${JWT_SECRET} \\
                        docker-compose up -d
                    """

                    sh "docker logout"

                    // Show deployment status
                    sh """
                        cd docker
                        echo "=== Docker Compose Status ==="
                        docker-compose ps
                        echo "=== Container Logs (last 20 lines) ==="
                        docker-compose logs --tail=20
                    """

                    echo "============================================"
                    echo "Docker Compose deployment completed successfully"
                    echo "============================================"
                    echo "Application URLs:"
                    echo "  Frontend:    http://localhost:${frontPort}"
                    echo "  Backend:     http://localhost:${backPort}"
                    echo "  Backend API: http://localhost:${frontPort}/api"
                    echo "  MySQL:       jdbc:mysql://localhost:${mysqlPort}/adeuxpas"
                    echo "============================================"
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
