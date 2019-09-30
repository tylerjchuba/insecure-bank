pipeline {
    agent none
    environment {
        BLACKDUCK_ACCESS_TOKEN  = credentials('jenkins-blackduck-access-token')
        PROTECODE_SC_PASSWORD   = credentials('jenkins-protecode-sc-password')
        DOCKER_LOGIN_PASSWORD   = credentials('jenkins-docker-login-password')
        POLARIS_ACCESS_TOKEN = credentials('jenkins-polaris-access-token')
    }
    stages {
        
      stage('Lightweight SCA') {
        agent { label 'maven-app' }
        steps {
          container('maven-with-wget') {
            sh 'curl -o detect.sh https://detect.synopsys.com/detect.sh'
            sh 'chmod +x detect.sh'
            sh './detect.sh \
                --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                --blackduck.api.token="MDVlYWEyODQtMzc5NS00NzVkLWJhN2MtN2M4YWY3ZmUwMjJiOjRmNjc0OWEyLWFiZjUtNDgwNS05ZjBjLTllNzJmNjVmYmNhNQ==" \
                --blackduck.trust.cert=true \
                --detect.project.name="CloudBeesInsecureBank" \
                --detect.tools="DETECTOR" \
                --detect.project.version.name="ALL_${BUILD_TAG}" \
                --detect.report.timeout=9000' 
          }
        }
      }

      stage('Parallel Build & Scan') {
          parallel {
              
                stage('Build App') {
                    agent { label 'maven-app' }
                    steps {
                      container('maven') {
                        sh 'mvn clean package'
                        stash includes: 'target/**', name: 'builtSources'
                      }
                    }
                }
              
                stage('SAST + SCA') {
                    agent { label 'maven-app' }
                    steps {
                      container('maven-with-wget') {
                        sh 'curl -o detect.sh https://detect.synopsys.com/detect.sh'
                        sh 'chmod +x detect.sh'
                        sh './detect.sh \
                            --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                            --blackduck.api.token="MDVlYWEyODQtMzc5NS00NzVkLWJhN2MtN2M4YWY3ZmUwMjJiOjRmNjc0OWEyLWFiZjUtNDgwNS05ZjBjLTllNzJmNjVmYmNhNQ==" \
                            --blackduck.trust.cert=true \
                            --detect.polaris.enabled=true \
                            --polaris.url="https://sipse.polaris.synopsys.com" \
                            --polaris.access.token="${POLARIS_ACCESS_TOKEN}" \
                            --detect.project.name="CloudBeesInsecureBank" \
                            --detect.tools="SIGNATURE_SCAN,BINARY_SCAN,POLARIS" \
                            --detect.project.version.name="All_${BUILD_TAG}" \
                            --detect.binary.scan.file.path="target/insecure-bank.war" \
                            --detect.blackduck.signature.scanner.paths=src/,target/ \
                            --detect.report.timeout=9000' 
                      }
                   }
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
            sh 'docker build -t cloudbees_detect_app:latest .'
            sh 'docker save -o /opt/blackduck/shared/target/cloudbees_detect_app.tar cloudbees_detect_app:latest'
          }
        }
      }
        
      stage('Scan') {
            parallel {
                stage('Container Image Scan') {
                    agent { label "docker-app" }
                    steps {
                        container('docker-with-detect') {
                            sh '/opt/blackduck/detect.sh \
                                    --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                                    --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
                                    --blackduck.trust.cert=true \
                                    --logging.level.com.synopsys.integration=DEBUG \
                                    --detect.project.name="CloudBeesInsecureBank" \
                                    --detect.tools="DOCKER,BINARY_SCAN" \
                                    --detect.binary.scan.file.path="/opt/blackduck/shared/target/cloudbees_detect_app.tar" \
                                    --detect.docker.image="cloudbees_detect_app:latest" \
                                    --detect.project.version.name="ALL_${BUILD_TAG}" \
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
                                --app="/opt/blackduck/shared/target/cloudbees_detect_app.tar" \
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
            sh 'docker tag cloudbees_detect_app:latest gautambaghel/cloudbees_detect_app:latest'
            sh 'docker push gautambaghel/cloudbees_detect_app:latest'
          }
        }
      }
          
  }
}
