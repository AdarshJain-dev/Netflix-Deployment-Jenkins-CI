
pipeline{
    agent any
    tools{
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        CD_REPO = "https://github.com/AdarshJain-dev/Netflix-Deployment-ArgoCD-CD.git"
        IMAGE_NAME = "adarshjain428/netflix"
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/AdarshJain-dev/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('Get Next Image Version') {
            steps {
                script {
                    sh '''
                    rm -rf cd-repo
                    git clone ${CD_REPO} cd-repo
                    cd cd-repo

                    CURRENT=$(grep image deployment-service.yml | awk -F:v '{print $2}')
                    NEXT=$((CURRENT + 1))

                    echo "v${NEXT}" > ../version.txt
                    '''
                    env.IMAGE_VERSION = sh(
                        script: "cat version.txt",
                        returnStdout: true
                    ).trim()
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                       sh """
                        docker build \
                          --build-arg TMDB_V3_API_KEY=3eb125d00454fb134d3de4025d2e0ac0 \
                          -t ${IMAGE_NAME}:${IMAGE_VERSION} .

                        docker push ${IMAGE_NAME}:${IMAGE_VERSION}
                        """
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image ${IMAGE_NAME}:${IMAGE_VERSION} > trivyimage.txt" 
            }
        }
        stage('Update CD Repo') {
            steps {
                script {
                    withCredentials([gitUsernamePassword(
                        credentialsId: 'git-cred',
                        gitToolName: 'Default'
                    )]) {
                        sh '''
                        cd cd-repo

                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_VERSION}|g" deployment-service.yml

                        git add deployment-service.yml
                        git commit -m "Deploy Netflix image ${IMAGE_VERSION}"
                        git push origin main
                        '''
                    }
                }
            }
        }
    }
}
