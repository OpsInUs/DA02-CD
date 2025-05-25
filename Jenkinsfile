pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_REPOSITORY = "vuhoabinhthachhoa"
        HELM_REPO_URL = "https://github.com/OpsInUs/DA02-HelmRepo.git"
        HELM_REPO_BRANCH = "main"
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Get the actual branch name from Jenkins environment or git
                    env.CURRENT_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    
                    // Clean branch name (remove origin/ prefix if present)
                    if (env.CURRENT_BRANCH.startsWith('origin/')) {
                        env.CURRENT_BRANCH = env.CURRENT_BRANCH.replace('origin/', '')
                    }
                    
                    def tagName = sh(script: "git tag --points-at HEAD 2>/dev/null | head -1 || echo ''", returnStdout: true).trim()
                    
                    if (tagName) {
                        env.TARGET_ENV = 'staging'
                        env.IMAGE_TAG = tagName
                    } else if (env.CURRENT_BRANCH == 'main' || env.CURRENT_BRANCH == 'master') {
                        env.TARGET_ENV = 'dev'
                        env.IMAGE_TAG = 'latest'
                    } else {
                        env.TARGET_ENV = 'dev-review'
                        env.IMAGE_TAG = env.GIT_COMMIT_SHORT
                    }
                    
                    echo "Branch: ${env.CURRENT_BRANCH} | Environment: ${env.TARGET_ENV} | Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Detect Changed Services') {
            steps {
                script {
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
                    
                    def changedServices = []
                    
                    try {
                        def changedFiles = sh(
                            script: "git diff --name-only HEAD~1 HEAD",
                            returnStdout: true
                        ).trim()
                        
                        if (changedFiles) {
                            def changedFilesList = changedFiles.split('\n')
                            echo "Changed files: ${changedFilesList.join(', ')}"
                            
                            services.each { service ->
                                def serviceChanged = changedFilesList.any { file -> 
                                    file.contains(service) || file.startsWith("${service}/")
                                }
                                if (serviceChanged) {
                                    changedServices.add(service)
                                }
                            }
                            
                            // Check for shared changes
                            def hasSharedChanges = changedFilesList.any { file ->
                                file == 'pom.xml' ||
                                file.startsWith('docker/') ||
                                file == 'Dockerfile' ||
                                file == 'Jenkinsfile'
                            }
                            
                            if (hasSharedChanges && changedServices.isEmpty()) {
                                changedServices = services
                            }
                        }
                        
                    } catch (Exception e) {
                        echo "Error detecting changes: ${e.message}"
                        changedServices = services
                    }
                    
                    if (changedServices.isEmpty()) {
                        changedServices = services
                    }
                    
                    env.CHANGED_SERVICES = changedServices.join(',')
                    echo "Services to build: ${changedServices.join(', ')}"
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    try {
                        def servicesToBuild = env.CHANGED_SERVICES.split(',')
                        echo "Starting to build services: ${servicesToBuild}"
                        
                        withCredentials([string(credentialsId: 'docker-hub-token', variable: 'DOCKER_HUB_TOKEN')]) {
                            // Use single quotes for better security
                            sh '''
                                echo "${DOCKER_HUB_TOKEN}" | docker login -u ${DOCKER_REPOSITORY} --password-stdin ${DOCKER_REGISTRY}
                            '''
                        }
                        
                        servicesToBuild.each { service ->
                            echo "Processing service: ${service}"
                            
                            // Check if directory exists
                            sh "if [ -d '${service}' ]; then echo 'Directory ${service} exists'; else echo 'Directory ${service} does NOT exist'; fi"
                            
                            dir(service) {
                                echo "Inside directory: ${service}"
                                sh 'pwd && ls -la'
                                
                                def mvnCmd = fileExists('./mvnw') ? './mvnw' : (fileExists('../mvnw') ? '../mvnw' : 'mvn')
                                echo "Using Maven command: ${mvnCmd}"
                                
                                if (mvnCmd.contains('mvnw')) {
                                    sh "chmod +x ${mvnCmd}"
                                }
                                
                                // Remove quiet flag to see build errors
                                sh "${mvnCmd} clean package -DskipTests"
                                
                                // Check if the artifact was created
                                sh "ls -la target/ || echo 'Target directory not found'"
                                
                                def imageName = "${env.DOCKER_REPOSITORY}/${service}:${env.IMAGE_TAG}"
                                echo "Building Docker image: ${imageName}"
                                
                                // Check if Dockerfile exists
                                sh "if [ -f '../docker/Dockerfile' ]; then echo 'Dockerfile exists'; else echo 'Dockerfile NOT found'; fi"
                                
                                sh """
                                    docker build -t ${imageName} -f ../docker/Dockerfile \\
                                        --build-arg ARTIFACT_NAME=target/${service}-3.4.1 \\
                                        --build-arg EXPOSED_PORT=8080 . || echo "Docker build failed"
                                    docker push ${imageName} || echo "Docker push failed"
                                """
                                echo "Built and pushed: ${imageName}"
                            }
                        }
                    } catch (Exception e) {
                        echo "Exception caught: ${e.getMessage()}"
                        echo "Stack trace: ${e.getStackTrace().join('\n')}"
                        throw e
                    }
                }
            }
        }

        stage('Update Helm Repository') {
            steps {
                script {
                    if(!tagName) {
                        echo "No tag found, skipping Helm repository update."
                        return
                    }
                    sh """
                        rm -rf helm-repo
                        git clone ${env.HELM_REPO_URL} helm-repo
                        cd helm-repo && git checkout ${env.HELM_REPO_BRANCH}
                    """
                    
                    sh "mkdir -p helm-repo/env/${env.TARGET_ENV}"
                    
                    def servicesToUpdate = env.CHANGED_SERVICES.split(',')
                    
                    servicesToUpdate.each { service ->
                        def helmServiceName = service.replace('spring-petclinic-', '')
                        
                        def yqAvailable = sh(script: "which yq >/dev/null 2>&1", returnStatus: true) == 0
                        
                        if (yqAvailable) {
                            sh """
                                cd helm-repo
                                yq eval -i '.services.${helmServiceName}.image.tag = "${env.IMAGE_TAG}"' env/${env.TARGET_ENV}/values.yaml
                            """
                        } else {
                            sh """
                                cd helm-repo
                                sed -i '/services:/,/^[[:space:]]*${helmServiceName}:/,/^[[:space:]]*image:/,/^[[:space:]]*tag:/ s/tag:.*/tag: "${env.IMAGE_TAG}"/' env/${env.TARGET_ENV}/values.yaml
                            """
                        }
                    }
                    
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            cd helm-repo
                            git config user.email "htkt004@gmail.com"
                            git config user.name "Tuyen572004"
                            
                            if ! git diff --quiet; then
                                git add env/${env.TARGET_ENV}/values.yaml
                                git commit -m "Update ${env.TARGET_ENV}: ${env.CHANGED_SERVICES} to ${env.IMAGE_TAG}"
                                git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/OpsInUs/DA02-HelmRepo.git ${env.HELM_REPO_BRANCH}
                                echo "Helm values updated"
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f || true'
            sh 'rm -rf helm-repo || true'
            cleanWs()
        }
    }
}