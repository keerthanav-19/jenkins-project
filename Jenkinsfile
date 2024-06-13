def label = "newpod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
containerTemplate(name: 'python', image: 'python:3.11', ttyEnabled: true, command: 'cat'),
containerTemplate(name: 'maven', image: 'maven:3.8-openjdk-11', ttyEnabled: true, command: 'cat'),
containerTemplate(name: 'curl', image: 'curlimages/curl:latest', ttyEnabled: true, command: 'cat'),
containerTemplate(name: 'kubectl', image: 'smesch/kubectl', ttyEnabled: true, command: 'cat', volumes: [secretVolume(secretName: 'kube-config', mountPath: '/root/.kube')]),
]){
    node(label) {
        def DOCKER_IMG_NAME = 'argocd-pythonapp'
        def VERSION = "${BUILD_ID}"
        def K8S_DEPLOYMENT_NAME = 'argocd-pythonapp-todo'
        def GITHUB_REPO_URL = "${env.GITHUB_REPO_URL}"
        def ARGOCD_AUTH_TOKEN = "${env.ARGOCD_AUTH_TOKEN}"
        def GITHUB_BRANCH = "${env.GITHUB_BRANCH}"
        def PERSONAL_ACCESS_TOKEN = "${env.PERSONAL_ACCESS_TOKEN}"

        stage('Checkout') {
            git branch: 'main', url: 'https://github.com/Keerthanachinnu/jenkins-project.git'
            sh 'ls'           
        }
        stage('Build and Test') {
            container('python') {
                sh 'python -m pip install --upgrade pip'
                sh 'pip install -r requirements.txt'
                sh 'python manage.py test'
            }
        } 
        stage('Sonarqube Analysis') {
            container('maven') {
                withCredentials([usernamePassword(credentialsId: 'sonar-creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh 'apt-get update'
                    sh 'apt-get install -y nodejs'

                    withSonarQubeEnv('SonarQube') {
                        def SONAR_HOST_URL='<YOUR SONAR URL>'
                        def scannerHome = tool 'SonarQube';
                        def sonarScannerCmd = "${scannerHome}/bin/sonar-scanner"
                        def SONAR_PROJECT_KEY = 'todo-python-app'
                        sh "${sonarScannerCmd} -Dsonar.login=$USERNAME -Dsonar.password=$PASSWORD -Dsonar.projectKey=${SONAR_PROJECT_KEY}"
                        sleep 30
                        def SONAR_PROJECT_NAME = 'todo-python-project'
                        sh '''
                            curl -k -u ${USERNAME}:${PASSWORD} "${SONAR_HOST_URL}/api/measures/component?metricKeys=critical_violations,coverage,major_violations,blocker_violations&component=${SONAR_PROJECT_NAME}" > sonar-result.json
                            cat sonar-result.json
                        '''
                    }
                }               
            }
        }
        stage('Build and Push Docker Image - Python App') {
            withCredentials([usernamePassword(credentialsId: 'docker-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                sh '''
                    docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                    docker build -t ${DOCKER_USERNAME}/${DOCKER_IMG_NAME}:${VERSION}
                    docker push ${DOCKER_USERNAME}/${DOCKER_IMG_NAME}:${VERSION}
                '''
            }
        }
        stage('Deployment - Python App') {
            container('kubectl') { 
                // Execute the command to get the deployment status
                def deploymentStatus = sh(script: "kubectl get deployment/${K8S_DEPLOYMENT_NAME}", returnStatus: true)
                // Check if the deployment exists
                sh "echo ${deploymentStatus}" 
                if (deploymentStatus == 0) {
                    // Deployment exists, execute deployment block
                    try {
                        String IMAGE = sh (script: "kubectl get --no-headers=true pods -l app=${K8S_DEPLOYMENT_NAME} -o custom-columns=:spec.containers[].image", returnStdout: true).trim()
                        sh "kubectl set image deployment/${K8S_DEPLOYMENT_NAME} ${K8S_DEPLOYMENT_NAME}=${CONT_REGISTRY}"
                        POD_NAME = sh (script: "kubectl get --no-headers=true pods -l app=${K8S_DEPLOYMENT_NAME} -o custom-columns=:metadata.name", returnStdout: true).trim()
                        def TAG_PRESENT = "${IMAGE}".contains("${VERSION}")
                        sh '''  
                            echo "POD_NAME - ${POD_NAME}"
                            echo "IMAGE - ${IMAGE}"
                            echo "TAG_PRESENT - ${TAG_PRESENT}"
                        ''' 
                        if (TAG_PRESENT) {
                            sh "kubectl delete pod ${POD_NAME}"
                        } else {
                            // ArgoCD commands for deployment
                            container('curl') {
                                // Download and install ArgoCD CLI
                                sh "curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v1.7.10/argocd-linux-amd64"
                                sh "chmod +x /usr/local/bin/argocd"
                                // ArgoCD commands for deployment
                        
                                sh '''
                                    APP_NAME=$(basename $GITHUB_REPO_URL .git)-$GITHUB_BRANCH
                                    argocd repo add --upsert $GITHUB_REPO_URL --username root --password $PERSONAL_ACCESS_TOKEN --plaintext --server <YOUR ARGOCD SERVER>
                                    argocd app create $APP_NAME --repo $GITHUB_REPO_URL --revision $GITHUB_BRANCH --path ./manifests/  --dest-server https://kubernetes.default.svc --kustomize-image myImage=$CONT_REGISTRY --server <YOUR ARGOCD SERVER> --auth-token=$ARGOCD_AUTH_TOKEN --plaintext --upsert                                      
                                    argocd app sync $APP_NAME --plaintext --prune --server <YOUR ARGOCD SERVER>
                                    argocd app set $APP_NAME --sync-policy automated --auto-prune --plaintext --server <YOUR ARGOCD SERVER>
                                '''    
                            } 
                        }
                        sleep 30
                        sh "kubectl get pods | grep argocd-pythonapp-todo"
                    } catch (e) {
                        container('curl') {
                            sh "ls -ll"
                            sh "curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v1.7.10/argocd-linux-amd64"
                            sh "chmod +x /usr/local/bin/argocd"
                            sh '''
                                APP_NAME=$(basename $GITHUB_REPO_URL .git)-$GITHUB_BRANCH
                                argocd repo add --upsert $GITHUB_REPO_URL --username root --password $PERSONAL_ACCESS_TOKEN --plaintext --server <YOUR ARGOCD SERVER>
                                argocd app create $APP_NAME --repo $GITHUB_REPO_URL --revision $GITHUB_BRANCH --path ./manifests/  --dest-server https://kubernetes.default.svc --kustomize-image myImage=$CONT_REGISTRY --server <YOUR ARGOCD SERVER> --auth-token=$ARGOCD_AUTH_TOKEN --plaintext --upsert                                      
                                argocd app sync $APP_NAME --plaintext --prune --server <YOUR ARGOCD SERVER>
                                argocd app set $APP_NAME --sync-policy automated --auto-prune --plaintext  --server <YOUR ARGOCD SERVER>                                    
                            '''        
                        }
                    }
                } else {
                    // Deployment does not exist, execute this block
                    sh 'ls -ll'
                    container('curl') {
                        sh 'curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v1.7.10/argocd-linux-amd64'
                        sh 'chmod +x /usr/local/bin/argocd'
                        sh '''
                            APP_NAME=$(basename $GITHUB_REPO_URL .git)-$GITHUB_BRANCH
                            argocd repo add --upsert $GITHUB_REPO_URL --username root --password $PERSONAL_ACCESS_TOKEN --plaintext --server <YOUR ARGOCD SERVER>
                            argocd app create $APP_NAME --repo $GITHUB_REPO_URL --revision $GITHUB_BRANCH --path ./manifests/  --dest-server https://kubernetes.default.svc --kustomize-image myImage=$CONT_REGISTRY --server <YOUR ARGOCD SERVER> --auth-token=$ARGOCD_AUTH_TOKEN --plaintext --upsert                                      
                            argocd app sync $APP_NAME --plaintext --prune --server <YOUR ARGOCD SERVER>
                            argocd app set $APP_NAME --sync-policy automated --auto-prune --plaintext  --server <YOUR ARGOCD SERVER>                                 
                        '''    
                    }  
                }
            }
        }
    }
}
