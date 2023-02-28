// change with your own server
def SERVER_DEST = "13.250.40.64"

pipeline{
    agent any
    options{
        buildDiscarder(logRotator(numToKeepStr: '5', daysToKeepStr: '5'))
        timestamps()
    }
    environment{
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        APP_NAME = "pastebin"
        APP_IMAGE_URL = "docker.io/tuyentd"
        IMAGE_NAME = "${APP_IMAGE_URL}/${APP_NAME}:${BUILD_NUMBER}"
        GIT_ACCESS_DATA = credentials('github_access_with_username_token')
    }

    stages{
        stage('Update Source Code') {
            steps {
                echo pwd()
                git credentialsId: 'githubCreds', url: 'https://github.com/dansolo1304/pastebin.git'
                }
            }

        stage('Build Application Image') {
            steps{
                withDockerRegistry(credentialsId: 'dockerioCreds', url: 'https://index.docker.io/v1/') {
                    sh '''docker build -t ${IMAGE_NAME} .'''
                    sh '''docker push ${IMAGE_NAME} '''
                }
                // replace docker-compose file
                sh '''  sed -i s~#IMAGE_NAME#~${IMAGE_NAME}~g ./docker-compose.yml && 
                        sed -i s~#APP_NAME#~${APP_NAME}~g ./docker-compose.yml
                    '''
                sh "cat ./docker-compose.yml"
            }
        }

        // Replace artifact and store in the deployment artifact git repo
        stage('Distribute Artifacts') {
            steps {
                dir("/tmp/centralize-deployment-artifacts") {
                git credentialsId: 'githubCreds', url: 'https://github.com/dansolo1304/centralize-deployment-artifacts.git'
                sh '''
                cp /var/lib/jenkins/workspace/devr-pastebin-demo-pipeline/docker-compose.yml . && 
                cat docker-compose.yml
                '''

                // push new changes to artifact repo
                sh "git config --global user.email \"tuyentd@yandex.com\""
                sh "git config --global user.name \"TuyenTD\""
                sh "git add --all ."
                sh "git commit -m \"Store deployment artifacts \""
                sh "git remote set-url origin https://${GIT_ACCESS_DATA}@github.com/dansolo1304/centralize-deployment-artifacts.git"
                sh "git pull origin master"
                sh "git push origin master"
                }
            }
        }
        // Pull the deployment artifacts and deploy into server
        // stage('Deploy Application in Same Server with jenkins') {
        //     steps {
        //         git credentialsId: 'githubCreds', url: 'https://github.com/dansolo1304/centralize-deployment-artifacts.git'
        //         echo pwd()
        //         sh '''
        //         docker-compose up -d
        //         '''
        //         }
        //     }

        stage('Deploy application to server using SSH') {
            steps {
                sshagent(['deployer-ssh-login']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -l deployer ${SERVER_DEST} "rm -rf centralize-deployment-artifacts-latest
                                                                                git clone https://${GIT_ACCESS_DATA}@github.com/dansolo1304/centralize-deployment-artifacts.git centralize-deployment-artifacts-latest
                                                                                cp -f centralize-deployment-artifacts-latest/docker-compose.yml centralize-deployment-artifacts/docker-compose.yml 
                                                                                cd centralize-deployment-artifacts
                                                                                URI="http://localhost" docker-compose up -d
                                                                                docker system prune -f
                                                                                "
                     """
                }
            }
        }


        stage('Verify application version') {
            steps {
                echo "Not yet implemet, need to make sure the app is running the correct version, before verifying browser accessing status"
                }
        }
        

        stage("Verify app using curl") {
            steps {
                script {
                    def APP_URL = "http://${SERVER_DEST}:8081/new"
                    def STATUS_CODE = sh(script: "curl -o /dev/null -s -w '%{http_code}' ${APP_URL}", returnStdout: true).trim()
                    if (STATUS_CODE.equals("200")) {
                        println "It works!"
                    } else {
                        currentBuild.result = 'FAILED'
                        println "Oops, something bad happened, please check the deployment!"
                    } 
                }
            }
        }
    }
}
