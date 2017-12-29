pipeline {
    agent {
        label "hornet-js"
    }

    environment {
        // Projet
        MODULE_GROUP="hornet-js"
        MODULE_ID="hornet-themes"

        BUILD_TIMESTAMP=sh(script: 'date +%Y%m%d%H%M%S', returnStdout:true).trim()
        VERSION_PACKAGE=sh(script:"npm run version --silent", returnStdout:true).trim()

        VERSION_RELEASE="${VERSION_PACKAGE}"
        VERSION_SNAPSHOT="${VERSION_RELEASE}-${BUILD_TIMESTAMP}"

        // Construction
        HORNETJSBUILDER_VERSION="latest"

        // Publication
        ARTIFACTORY_URL = ""
        REPOSITORY_GROUP="hornet"
        REPOSITORY_SNAPSHOT = "${REPOSITORY_GROUP}-snapshot"
        REPOSITORY_RELEASE = "${REPOSITORY_GROUP}-release"
        REPOSITORY_NPM_SNAPSHOT = "${REPOSITORY_GROUP}-npm-snapshot"
        REPOSITORY_NPM_RELEASE = "${REPOSITORY_GROUP}-npm-release"

        // Qualité
        SONAR_HOST = ""
        SONAR_CREDENTIALS_KEY = "hornet_cq_at_sonar01"
        SONAR_SCANNER_CLI = "3.0.3.778"
    }

    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: "20", artifactNumToKeepStr: "1", daysToKeepStr: "90", numToKeepStr: "20"))
        disableConcurrentBuilds()
    }

    stages {
        stage("Environnement") {
            steps {
                echo sh(script: "env|sort", returnStdout: true)
            }
            post {
                success {
                    echo "[SUCCESS] Success to print Environnement"
                }
                failure {
                    echo "[FAILURE] Failed to print Environnement"
                }
            }
        }

        stage("Install Builder") {
            steps {
                withNPM(npmrcConfig: "npmrc_hornet") {
                    sh '''
						echo "Install hornet-js-builder@$HORNETJSBUILDER_VERSION"
						echo "${HORNETJSBUILDER_VERSION}" > hb_version
						bash hbw.sh --version
                       '''
                }
            }
            post {
                success {
                    echo "[SUCCESS] Success to install builder"
                }
                failure {
                    echo "[FAILURE] Failed to install builder"
                }
            }
        }

        stage("Handle Snaphot version") {
            when {
				anyOf {
	                branch "develop"
				}
            }
            steps {
                    sh "bash hbw.sh versions:set --versionFix=${VERSION_SNAPSHOT}"
            }
            post {
                success {
                    echo "[SUCCESS] Success to handle Snapshot version"
                }
                failure {
                    echo "[FAILURE] Failed to handle Snapshot version"
                }
            }
        }

        stage("Build") {
            steps {
                sh "bash hbw.sh clean package"

            }
            post {
                success {
                    echo "[SUCCESS] Success to build"
                    archive "target/*.zip"
                }
                failure {
                    echo "[FAILURE] Failed to build"
                }
            }
        }

        stage("Publish snapshot") {
            when {
                branch "develop"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "hornet_ci_at_artifactory", passwordVariable: "pwd_ci", usernameVariable: "user_ci")]) {
                    script {
                        def artifactory = Artifactory.newServer url: "${ARTIFACTORY_URL}", username: "${user_ci}", password: "${pwd_ci}"
                        def uploadSpec = """{
                            "files": [
                            {
                                "pattern": "target/*.zip",
                                "target": "${REPOSITORY_SNAPSHOT}/${MODULE_GROUP}/${MODULE_ID}/${VERSION_SNAPSHOT}/"

                            }
                        ]
                        }"""
                        artifactory.upload(uploadSpec)

                    }
                }
                archiveArtifacts artifacts: "target/*.zip", fingerprint: true
            }
            post {
                success {
                    echo "[SUCCESS] Success to Publish snapshot"
                }
                failure {
                    echo "[FAILURE] Failed to Publish snapshot"
                }
            }
        }

        stage("Publish NPM snapshot") {
            when {
                branch "develop"
            }
            steps {
                sh "bash hbw.sh publish --publish-registry ${ARTIFACTORY_URL}/api/npm/${REPOSITORY_NPM_SNAPSHOT}"
            }
            post {
                success {
                    echo "[SUCCESS] Success to Publish snapshot"
                }
                failure {
                    echo "[FAILURE] Failed to Publish snapshot"
                }
            }
        }

        stage("Publish release") {
            when {
                branch "master"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "hornet_ci_at_artifactory", passwordVariable: "pwd_ci", usernameVariable: "user_ci")]) {
                    script {
                        def artifactory = Artifactory.newServer url: "${ARTIFACTORY_URL}", username: "${user_ci}", password: "${pwd_ci}"
                        def uploadSpec = """{
                            "files": [
                            {
                                "pattern": "target/*.zip",
                                "target": "${REPOSITORY_RELEASE}/${MODULE_GROUP}/${MODULE_ID}/${VERSION_RELEASE}/"

                            }
                        ]
                        }"""
                        artifactory.upload(uploadSpec)

                    }
                }
                archiveArtifacts artifacts: "target/*.zip", fingerprint: true
            }
            post {
                success {
                    echo "[SUCCESS] Success to Publish release"
                }
                failure {
                    echo "[FAILURE] Failed to Publish release"
                }
            }
        }

        stage("Publish NPM release") {
            when {
                branch "master"
            }
            steps {
                sh "bash hbw.sh publish --publish-registry ${ARTIFACTORY_URL}/api/npm/${REPOSITORY_NPM_RELEASE}"
            }
            post {
                success {
                    echo "[SUCCESS] Success to Publish release"
                }
                failure {
                    echo "[FAILURE] Failed to Publish release"
                }
            }
        }


        stage("Quality") {
            environment {
                SONAR_CREDENTIALS = credentials("${SONAR_CREDENTIALS_KEY}")
            }
            when {
                anyOf {
                    branch "develop"
                    branch "master"
                }
            }
            steps {
                script {
                  scannerHome = tool "SonarQube Scanner $SONAR_SCANNER_CLI"
                }

                sh '''
                    echo "
                    sonar.host.url=${SONAR_HOST}
                    sonar.login=${SONAR_CREDENTIALS_PSW}
                    sonar.projectKey=${MODULE_GROUP}:${MODULE_ID}
                    sonar.projectName=${MODULE_ID}
                    sonar.projectVersion=${VERSION_SNAPSHOT}
                    sonar.sourceEncoding=UTF-8
                    sonar.sources=./src
                    sonar.exclusions=**/node_modules/**,**/*.spec.ts

                    " > sonar-project.properties
                '''
               sh "${scannerHome}/bin/sonar-scanner"
            }

            post {
                success {
                    echo "[SUCCESS] Success to run Quality"
                }
                failure {
                    echo "[FAILURE] Failed to run Quality"
                }
            }
        }
    }
}
