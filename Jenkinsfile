pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKER_TAG',
            defaultValue: 'latest',
            description: 'Tag for the Docker image'
        )
    }

    tools {
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool('sonar-scanner')
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/chimdi247/Multi-Tier-BankApp-mysql-java-jenkins-CI.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    """
                }
            }
        }

      //  stage('Quality Gate') {
        //    steps {
          //      script {
            //        waitForQualityGate abortPipeline: false,
                       //                credentialsId: 'sonar-token'
             //   }
        //    }
    //    }

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'maven-settings-chimdi',
                    jdk: 'jdk21',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn clean deploy -DskipTests'
                }
            }
        }

        stage('Docker') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t chimdi247/bankapp:${params.DOCKER_TAG} ."
                        sh "docker push chimdi247/bankapp:${params.DOCKER_TAG}"
                    }
                }
            }
        }

        stage('Update YAML Manifest in Other Repo') {
            steps {
                script {
                    withCredentials([
                        gitUsernamePassword(
                            credentialsId: 'git-cred',
                            gitToolName: 'Default'
                        )
                    ]) {

                        sh """
                            git clone https://github.com/chimdi247/Multi-Tier-BankApp-Java-mysql-Jenkin-CD.git
                            cd Multi-Tier-BankApp-Java-mysql-Jenkin-CD

                            ls -l bankapp

                            sed -i "s|image: chimdi247/bankapp:.*|image: chimdi247/bankapp:${params.DOCKER_TAG}|" \
                            bankapp/bankapp-ds.yml
                        """

                        sh '''
                            echo "Updated YAML file contents:"
                            cat  Multi-Tier-BankApp-Java-mysql-Jenkin-CD/bankapp/bankapp-ds.yml
                        '''

                        sh '''
                            cd  Multi-Tier-BankApp-Java-mysql-Jenkin-CD
                            git config user.email "office@chimdi247@gmail.com"
                            git config user.name "chimdi247"
                        '''

                        sh """
                            cd  Multi-Tier-BankApp-Java-mysql-Jenkin-CD
                            git add bankapp/bankapp-ds.yml
                            git commit -m "Update image tag to ${params.DOCKER_TAG}" || true
                            git push origin main
                        """
                    }
                }
            }
        }
    }
}

