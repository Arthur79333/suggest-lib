pipeline {
    agent any

    triggers {
        pollSCM '* * * * *'
    }

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        GIT_CRED_ID = 'gitlab-suggest'
    }

    stages {
        stage('Version calculation') {
            when {
                branch 'release/*'
            }

            steps {
                script {
                    sshagent(credentials: ["${GIT_CRED_ID}"]) {
                        sh 'git fetch --tags'

                        // Get the latest tag if it exists
                        def latestTag = sh(script: 'git tag -l --merge | sort -V | tail -n 1', returnStdout: true).trim()
                        echo "LATEST_TAG: ${latestTag}"

                        if (latestTag) {
                            def (major, minor, patch) = latestTag.tokenize('.')
                            patch = patch.toInteger() + 1
                            CALCULATED_VERSION = "${major}.${minor}.${patch}"
                        } else {
                            def branchVersion = BRANCH_NAME.split('/')[1]
                            CALCULATED_VERSION = "${branchVersion}.1"
                        }

                        echo "CALCULATED_VERSION: ${CALCULATED_VERSION}"
                    }
                }
            }
        }

        stage('Maven') {
            agent {
                docker {
                    image 'maven:3-eclipse-temurin-8'
                    args '--network jenkins_network'
                    //args '-v $M2_CACHE_VOL:/root/.m2 --network jenkins_network'

                }
            }

            environment {
                SETTINGS_XML_ID = 'jenkins-artifactory-settings-xml'
                SNAPSHOT_REPO_ID = 'snapshots'
                RELEASE_REPO_ID = 'central'
            }

            stages {
                stage('Compile') {
                    steps {
                        sh 'mvn compile'
                    }
                }

                stage('Test') {
                    steps {
                        sh 'mvn verify'
                    }
                }

                stage('Publish') {
                    when {
                        anyOf {
                            branch 'main'
                            branch 'release/*'
                        }
                    }

                    steps {
                        configFileProvider([configFile(fileId: 'jenkins-settings', variable: 'MVN_SETTINGS')]) {
                            script {
                                if (BRANCH_NAME == 'main') {
                                    sh "mvn deploy -s ${MVN_SETTINGS} -DrepositoryId=${SNAPSHOT_REPO_ID}"
                                } else {
                                    sh "mvn versions:set -DnewVersion=${CALCULATED_VERSION}"
                                    sh "mvn deploy -s ${MVN_SETTINGS} -DrepositoryId=${RELEASE_REPO_ID}"
                                }
                            }
                        }
                    }
                }
            }

            post {
                always {
                    sh 'mvn clean'
                }
            }
        }

        stage('Git Tag & Clean') {


            when{
                branch "release/*"
            }


            steps {
                script {
                    sshagent(credentials: ["${GIT_CRED_ID}"]) {
                        sh """
                            git reset --hard
                            git tag ${CALCULATED_VERSION}
                            git push origin ${CALCULATED_VERSION}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}