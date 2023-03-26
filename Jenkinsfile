pipeline {

    agent any

    tools {
        maven 'maven-3.9.1'
    }

    environment {
        NEXUS_VERSION = 'nexus3'
        NEXUS_PROTOCOL = 'http'
        NEXUS_URL = '172.31.43.212:8081'
        NEXUS_REPO_NAME = 'vprofile-releases'
        NEXUS_CREDENTIAL_ID = 'nexus-credentials'
        NEXUS_GRP_REPO = 'vprofile-group'

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

        stage ('Upload to Nexus') {
            steps {
                script {
                    echo 'Uploading to Nexus Artifactory'
                    pom = readMavenPom file: "pom.xml";

                    nexusArtifactUploader artifacts: [
                        [artifactId: 'pom.aritfactId',
                         classifier: '',
                         file: "target/${pom.artifactId}",
                         type: 'pom.packaging']
                         ],
                         credentialsId: NEXUS_CREDENTIAL_ID, 
                         groupId: 'pom.groupId',
                         nexusUrl: NEXUS_URL,
                         nexusVersion: NEXUS_VERSION,
                         protocol: NEXUS_PROTOCOL,
                         repository: NEXUS_REPO_NAME,
                         version: APP_VERSION

                }
            }
        }        

        stage("commit version update"){
            steps{
                script{
                    sshagent(credentials: ['GitHub-SSH']) {
                        // sh "git remote set-url origin https://github.com/Vekeleme-Projects/ci-cd-projects.git"
                        // sh 'git add .'
                        // sh 'git commit -m "ci: version bump"'
                        // sh 'git push origin HEAD:main'
                        
                    }
                }
            }
        }       

    }



}
