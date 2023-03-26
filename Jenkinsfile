pipeline {

    agent any

    tools {
        maven 'maven-3.9.1'
    }

    environment {
        NEXUS_VERSION = 'nexus3'
        NEXUS_PROTOCOL = 'http'
        NEXUS_URL = '3.141.195.48:8081'
        NEXUS_REPO_NAME = 'vprofile-releases/'
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
                    env.APP_VERSION = "$version"
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
                        [artifactId: pom.artifactId,
                         classifier: '',
                         file: "target/${pom.artifactId}-${APP_VERSION}.war",
                         type: pom.packaging]
                         ],
                         credentialsId: NEXUS_CREDENTIAL_ID, 
                         groupId: pom.groupId,
                         nexusUrl: NEXUS_URL,
                         nexusVersion: NEXUS_VERSION,
                         protocol: NEXUS_PROTOCOL,
                         repository: NEXUS_REPO_NAME,
                         version: APP_VERSION

                }
            }
        }

        stage("Copy Files to Ansible Server"){
            steps{
                script{
                    sshagent(['ansible-ssh-credentials']) {
                        sh "scp -o StrictHostKeyChecking=no ansible/* ubuntu@3.135.248.204:/home/ubuntu/ansible/"

                        withCredentials([sshUserPrivateKey(credentialsId: 'server-ssh-key', keyFileVariable: 'keyfile', usernameVariable: 'ubuntu')]) {
                            sh 'scp ${keyfile} ubuntu@3.135.248.204:~/.ssh/ssh-key.pem'
                        }

                    }
                }
            }
        }


        stage ("Execute Ansible playbook") {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-credentials', keyFileVariable: 'identity', passphraseVariable: '',  usernameVariable: 'ubuntu')]) {

                        def remote = [:]
                        remote.name = "ubuntu"
                        remote.host = "3.135.248.204"
                        remote.allowAnyHosts = true
                        
                        remote.user = user
                        remote.identityFile = identity

                        sh "ssh -i ${identity} ubuntu@3.135.248.204 'ls -la'"
                        //sshCommand remote: remote, command: 'ls -la'
                    }
                }
            }
        }


        // stage("commit version update"){
        //     steps{
        //         script{
        //             withCredentials([string(credentialsId: 'git-credentials', variable: '')]) {
        //                 sh "git remote set-url origin https://github.com/Vekeleme-Projects/ci-cd-projects.git"
        //                 sh 'git add .'
        //                 sh 'git commit -m "ci: version bump"'
        //                 sh 'git push origin HEAD:main'
                        
        //             }
        //         }
        //     }
        // }       

    }



}
