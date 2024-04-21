pipeline {
    agent any
    environment {
        CONTAINER_NAME = "calculator"
        ENV_NAME = getEnvName(env.BRANCH_NAME)
        CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
        HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
        EMAIL_RECIPIENTS = "tbagor23@gmail.com"
    }

    stages {
            // stage('Initialize') {
            //     def dockerHome = tool 'DockerLatest'
            //     def mavenHome = tool 'MavenLatest'
            //     def javaHome = tool 'JavaLatest'
            //     env.PATH = "${dockerHome}/bin:${javaHome}/bin:${mavenHome}/bin:${env.PATH}"
            // }

            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('Build with test') {
                steps {
                    sh "mvn clean install"
                }
            }

            stage('Sonarqube Analysis') {
                steps {
                    withSonarQubeEnv('SonarQubeLocalServer') {
                        sh " mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
                    }
                    timeout(time: 1, unit: 'MINUTES') {
                        def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }

            stage("Image Prune") {
                steps {
                    imagePrune(CONTAINER_NAME)
                }
            }
            

            stage('Image Build') {
                steps {
                    imageBuild(CONTAINER_NAME, CONTAINER_TAG)
                }
            }
            

            stage('Push to Docker Registry') {
                steps {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
                    }
                }
            }

            stage('Run App') {
                steps {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        runApp(CONTAINER_NAME, CONTAINER_TAG, USERNAME, HTTP_PORT, ENV_NAME)
                    }
                }
            }
    }
}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName"
    } catch (ignored) {
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag  -t $containerName --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword) {
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def runApp(containerName, tag, dockerHubUser, httpPort, envName) {
    sh "docker pull $dockerHubUser/$containerName"
    sh "docker run --rm --env SPRING_ACTIVE_PROFILES=$envName -d -p $httpPort:$httpPort --name $containerName $dockerHubUser/$containerName:$tag"
    echo "Application started on port: ${httpPort} (http)"
}

def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}

String getEnvName(String branchName) {
    if (branchName == 'main') {
        return 'prod'
    }
    return (branchName == 'ready') ? 'uat' : 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'main') {
        return '9999'
    }
    return (branchName == 'ready') ? '8888' : '8090'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'main') {
        return buildNumber + '-stable'
    }
    return buildNumber + '-unstable'
}
