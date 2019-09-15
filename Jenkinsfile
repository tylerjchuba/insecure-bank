pipeline {
    agent none
    environment {
        BLACKDUCK_ACCESS_TOKEN  = credentials('jenkins-blackduck-access-token')
        PROTECODE_SC_PASSWORD   = credentials('jenkins-protecode-sc-password')
        DOCKER_LOGIN_PASSWORD   = credentials('jenkins-docker-login-password')
    }
    stages {

      stage('Build') {
        agent { label 'maven-app' }
        steps {
          container('maven') {
            sh 'mvn clean package'
            stash includes: 'target/**', name: 'builtSources'
          }
        }
      }

      stage('Save') {
        agent { label 'docker-app' }
        when {
          expression {
            currentBuild.result == null || currentBuild.result == 'SUCCESS'
          }
        }
        steps {
          container('docker-with-detect') {
            unstash 'builtSources'
            sh 'mkdir -p /opt/blackduck/shared/target/'
            sh 'docker build -t cloudbees_insecure_bank:latest .'
            sh 'docker save -o /opt/blackduck/shared/target/cloudbees_insecure_bank.tar cloudbees_insecure_bank:latest'
          }
        }
      }

      stage('Scan') {
            parallel {
                stage('Docker Inspector') {
                    agent { label "docker-app" }
                    steps {
                        container('docker-with-detect') {
                            sh '/opt/blackduck/detect.sh \
                                    --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                                    --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
                                    --blackduck.trust.cert=true \
                                    --logging.level.com.synopsys.integration=DEBUG \
                                    --detect.project.name="CloudBeesDucky" \
                                    --detect.tools="DOCKER" \
                                    --detect.docker.image="cloudbees_insecure_bank:latest" \
                                    --detect.project.version.name="DOCKER_${BUILD_TAG}" \
                                    --detect.risk.report.pdf=true \
                                    --detect.report.timeout=9000 \
                                    --detect.docker.passthrough.imageinspector.service.url="http://blackduck-imageinspector-alpine.blackduck-imageinspector" \
                                    --detect.docker.passthrough.shared.dir.path.local="/opt/blackduck/shared/" \
                                    --detect.docker.passthrough.shared.dir.path.imageinspector="/opt/blackduck/shared" \
                                    --detect.docker.passthrough.imageinspector.service.start=false'
                        }
                    }
                    post {
                        always {
                            stash includes: '**/*.pdf', name: 'detectReport'
                        }
                    }
                }
                stage('Black Duck Binary Analysis') {
                    agent { label "python-app" }
                    steps {
                        container('python') {
                            sh 'python /opt/blackduck/bdba-pdf.py \
                                --app="/opt/blackduck/shared/target/cloudbees_insecure_bank.tar" \
                                --protecode-host="protecode-sc.com" \
                                --protecode-username="gautamb@synopsys.com" \
                                --protecode-password="${PROTECODE_SC_PASSWORD}" \
                                --protecode-group="Duck Binaries"'
                            sh 'find . -type f -iname "*.pdf" -exec tar -cf synopsys_scan_results.tar "{}" +'
                        }
                    }
                    post {
                        always {
                            stash includes: '**/*.pdf', name: 'bdbaReport'
                        }
                    }
                }
            }
        }

      stage('Publish') {
        agent { label 'docker-app' }
        steps {
          container('docker-with-detect') {
            unstash 'detectReport'
            unstash 'bdbaReport'
            sh 'find . -type f -iname "*.pdf" -exec tar -cf synopsys_scan_results.tar "{}" +'
            archiveArtifacts artifacts: '**/*.tar', fingerprint: true, onlyIfSuccessful: true
              sh 'cat my_password.txt | docker login --username gautambaghel --password ${DOCKER_LOGIN_PASSWORD}'
            sh 'docker tag cloudbees_insecure_bank:latest gautambaghel/cloudbees_insecure_bank:latest'
            sh 'docker push gautambaghel/cloudbees_insecure_bank:latest'
          }
        }
      }

  }
}
