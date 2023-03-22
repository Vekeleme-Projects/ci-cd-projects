pipeline {

    agent any

    tools {
        maven 'maven-3.9.1'
    }

    environment {
        AWS_ECR_REPO_NAME = "524360703326.dkr.ecr.us-east-2.amazonaws.com"
        AWS_REGION = "us-east-2"
        DOCKER_IMAGE_NAME = "java-maven-app"
        IMAGE_TAG = APP_VERSION
        GIT_REPO_URL = "git@github.com:Vekeleme-Projects/ci-projects.git"
        


    }

    stages {

        stage("version increment"){
            steps{
                script{
                    sh "mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} versions:commit"
                    
                    def matchVersion = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matchVersion[0][1]
                    env.APP_VERSION = "$version-$BUILD_NUMBER"
                }
            }
        } 

        stage ('test') {
            when {
                expression {
                    params.executeTests == true
                }
            }
            steps {
                script {
                    echo 'Testing application'
                    sh 'mvn test'
                }
            }
        }

        stage ('build jar') {
            steps {
                script {
                    echo 'Building jar package'
                    sh 'mvn clean package'
                }
            }
        }

        stage ('SonarQube Code Analysis') {
            environment{
                sonarScanner = tool 'sonar-4.8.0'
            }
            steps {
                script {
                    echo 'Performing code analysis'
                    withSonarQubeEnv('sonar-server') {
                        sh 'mvn sonar:sonar'
                    }
                    timeout(time: 5, unit: 'MINUTES'){
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage("Build Docker image"){
            steps {
                script{
                    echo "Building docker Image"
                    docker build -t "${DOCKER_REPO_NAME}/${DOCKER_IMAGE_NAME}:${IMAGE_TAG}" .
                }
            }
        }    

        stage("Push Docker image To ECR"){
            steps {
                script{
                    echo "Pushing docker Image"
                    withCredentials([aws(accessKeyVariable:'AWS_ACCESS_KEY_ID',credentialsId:'aws-credentials',secretKeyVariable:'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${DOCKER_REPO_NAME}"
                        sh "docker push ${DOCKER_REPO_NAME}/${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }   

        stage("commit version update"){
            steps{
                script{
                    sshagent(credentials: ['GitHub-SSH']) {
                        sh "git remote set-url origin ${GIT_REPO_URL}"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:main'
                        
                    }
                }
            }
        }       

    }

}
