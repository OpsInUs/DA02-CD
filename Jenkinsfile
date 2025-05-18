pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_REPOSITORY = "vuhoabinhthachhoa"
        GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        HELM_REPO_URL = "https://github.com/OpsInUs/DA02-HelmRepo.git"
        HELM_REPO_BRANCH = "main"
    }

    stages {
        stage('Set Environment and Tag') {
            steps {
                script {
                    // Check if this is a tagged commit
                    def gitTag = sh(script: "git tag --points-at HEAD", returnStdout: true).trim()
                    def currentBranch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    
                    if (gitTag) {
                        env.TARGET_ENV = 'staging'
                        env.IMAGE_TAG = gitTag
                        env.UPDATE_HELM = 'true'
                        echo "Tagged commit '${gitTag}': Building for staging environment"
                    } else if (currentBranch == 'main' || currentBranch == 'master') {
                        env.TARGET_ENV = 'dev'
                        env.IMAGE_TAG = 'latest'
                        env.UPDATE_HELM = 'false'
                        echo "Main branch commit: Building for dev environment"
                    } else {
                        env.TARGET_ENV = 'dev-review'
                        env.IMAGE_TAG = env.GIT_COMMIT_SHORT
                        env.UPDATE_HELM = 'false'
                        echo "Feature branch commit: Building for dev-review environment"
                    }
                    
                    echo "Target Environment: ${env.TARGET_ENV} | Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Identify Changed Services') {
            steps {
                script {
                    def baseCommit
                    def services = [
                        'spring-petclinic-admin-server',
                        'spring-petclinic-api-gateway',
                        'spring-petclinic-config-server',
                        'spring-petclinic-customers-service',
                        'spring-petclinic-discovery-server',
                        'spring-petclinic-genai-service',
                        'spring-petclinic-vets-service',
                        'spring-petclinic-visits-service'
                    ]
                    
                    try {
                        // Determine base commit for comparison
                        if (currentBranch == 'main' || currentBranch == 'master') {
                            baseCommit = sh(script: "git rev-parse HEAD~1", returnStdout: true).trim()
                        } else {
                            baseCommit = sh(script: "git merge-base origin/main HEAD || git rev-parse HEAD~1", returnStdout: true).trim()
                        }
                        
                        // Get changed files
                        def changedFiles = sh(script: "git diff --name-only ${baseCommit} HEAD", returnStdout: true).trim()
                        def changedServices = []
                        
                        services.each { service ->
                            if (changedFiles.contains(service)) {
                                changedServices.add(service)
                                echo "Changed service: ${service}"
                            }
                        }
                        
                        if (changedServices.isEmpty()) {
                            echo "No specific services changed. Building all services."
                            changedServices = services
                        }
                        
                        env.CHANGED_SERVICES = changedServices.join(',')
                        env.SERVICES_TO_BUILD = changedServices // Store as environment variable
                        
                        echo "Services to build: ${env.CHANGED_SERVICES}"
                    } catch (Exception e) {
                        echo "Error detecting changed services: ${e.message}. Building all services."
                        env.CHANGED_SERVICES = services.join(',')
                        env.SERVICES_TO_BUILD = services.join(',')
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    def servicesToBuild = env.CHANGED_SERVICES.split(',')
                    
                    withCredentials([string(credentialsId: 'docker-hub-token', variable: 'DOCKER_HUB_TOKEN')]) {
                        sh 'echo "${DOCKER_HUB_TOKEN}" | docker login -u ${DOCKER_REPOSITORY} --password-stdin ${DOCKER_REGISTRY}'
                    }
                    
                    for (service in servicesToBuild) {
                        echo "Building ${service} with tag: ${env.IMAGE_TAG}"
                        buildAndPushService(service)
                    }
                }
            }
        }

        stage('Update Helm Values for Staging') {
            when {
                expression { return env.UPDATE_HELM == 'true' }
            }
            steps {
                script {
                    echo "Updating Helm values for staging environment with tag: ${env.IMAGE_TAG}"
                    updateHelmRepository()
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            sh 'docker system prune -f || true'
        }
        success {
            echo "Build successful: ${currentBuild.fullDisplayName}"
            echo "Environment: ${env.TARGET_ENV} | Image Tag: ${env.IMAGE_TAG}"
        }
        failure {
            echo "Build failed: ${currentBuild.fullDisplayName}"
        }
    }
}

// Helper function to build and push a service
def buildAndPushService(String service) {
    dir(service) {
        // Prepare Maven
        if (fileExists('./mvnw')) {
            sh "chmod +x ./mvnw"
            sh "./mvnw clean package -DskipTests"
        } else if (fileExists('../mvnw')) {
            sh "chmod +x ../mvnw"
            sh "../mvnw clean package -DskipTests"
        } else {
            sh "mvn clean package -DskipTests"
        }
        
        // Build and push Docker image
        sh """
            docker build \\
                -t ${DOCKER_REPOSITORY}/${service}:${env.IMAGE_TAG} \\
                -f ../docker/Dockerfile \\
                --build-arg ARTIFACT_NAME=target/${service}-3.4.1 \\
                --build-arg EXPOSED_PORT=8080 .

            docker push ${DOCKER_REPOSITORY}/${service}:${env.IMAGE_TAG}
        """
    }
}

// Helper function to update Helm repository
def updateHelmRepository() {
    sh "git clone ${HELM_REPO_URL} helm-repo"
    sh "mkdir -p helm-repo/env/${env.TARGET_ENV}"
    
    def services = env.CHANGED_SERVICES.split(',')
    for (service in services) {
        def helmServiceName = service.replace('spring-petclinic-', '')
        sh "yq e -i '.services.${helmServiceName}.image.tag = \"${env.IMAGE_TAG}\"' helm-repo/env/${env.TARGET_ENV}/values.yaml || echo 'YQ command failed'"
    }
    
    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
        sh """
            cd helm-repo
            git config --global user.email "htkt004@gmail.com"
            git config --global user.name "Tuyen572004"
            
            git config credential.helper store
            echo "https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com" > ~/.git-credentials
            
            git add env/${env.TARGET_ENV}/values.yaml
            git commit -m "Update ${env.TARGET_ENV} environment with ${env.IMAGE_TAG} tag for ${env.CHANGED_SERVICES}" || echo "No changes to commit"
            git push origin ${HELM_REPO_BRANCH}
            
            rm ~/.git-credentials
        """
    }
}