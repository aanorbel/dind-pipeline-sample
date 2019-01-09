pipeline {
    environment {
        registry = '192.168.33.11/openshift/jenkins-spring-build'
        registryCredential = 'openshift-pusher'
        dockerImage = ''
        latestDockerImage = ''
        repositoryName = 'libs-snapshot-local'
        moduleName = 'demo-openshift'
        def scannerHome = tool 'sonarqube'
        def uploadSpec = """{
             "files": [
               {
                 "pattern": "build/libs/*.jar",
                 "target": "${repositoryName}/com/sumelongenterprise/${moduleName}/0.0.1.BUILD-SNAPSHOT/${moduleName}.jar"
               }
             ]
            }"""
    }
    agent any
    stages {
        stage('SonarQube analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        stage("Compile") {
            steps {
                sh "./gradlew clean compileJava"
            }
        }

        stage("Package") {
            steps {
                sh "./gradlew build"
            }
        }

        stage("Publish to Artifacatory") {
            steps {
                script {

                    def server = Artifactory.server('artifactory')
                    server.bypassProxy = true
                    def buildInfo = server.upload spec: uploadSpec

                    server.publishBuildInfo buildInfo
                }
            }
        }

        stage("Docker build") {
            steps {
                script {
                    latestDockerImage = docker.build(registry)
                    dockerImage = docker.build("${registry}:${env.BUILD_ID}")
                }
            }
        }

        stage("Docker push") {
            steps {

                script {
                    docker.withRegistry('http://192.168.33.11', registryCredential) {
                        dockerImage.push()
                        latestDockerImage.push()
                    }
                }
            }
        }
        stage("Deploy to staging") {
            steps {
                sh "echo 'Deploy to staging'"
                script {
                    openshift.withCluster('openshift') {
                        openshift.withProject() {

                            // Output the url of the currently selected cluster
                            echo "Using project ${openshift.project()} in cluster with url ${openshift.cluster()}"

                            if (openshift.selector("dc", "jenkins-spring-build").exist()) {
                                print 'deployment config exists'
                            } else {
                                print 'deployment config does not exists'

                            }
                        }
                    }
                    openshiftDeploy apiURL: "${env.OPENSHIFT_CLUSTER_URL}", authToken: "${env.OPENSHIFT_JENKINS_TOKEN}", depCfg: 'jenkins-spring-build', namespace: 'development', verbose: 'true', waitTime: '2', waitUnit: 'min'
                }
            }
        }

        stage("Acceptance test") {
            steps {
                sh "echo 'Acceptance test'"

            }
        }
    }
    post {
        always {
            sh "echo 'Acceptance test'"
        }
    }
}
