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
          slackSend color: 'good', message: "${env.BUILD_TIMESTAMP} Starting SIG Pipeline on ${env.JOB_NAME}"
          slackSend color: 'good', message: "Performing Lightweight SCA with Blackduck"
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
                      slackSend color: 'good', message: "Building Maven appliction"
                      container('maven') {
                        sh 'mvn clean package'
                        stash includes: 'target/**', name: 'builtSources'
                        stash includes: 'target/insecure-bank.war', name: 'warfile'
                      }
                    }
                }

                stage('SAST + SCA') {
                    agent { label 'maven-app' }
                    steps {
                      slackSend color: 'good', message: "Running Coverity and Polaris"
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
                        slackSend color: 'good', message: "Running Container Scan"
                        container('docker-with-detect') {
                            sh '/opt/blackduck/detect.sh \
                                    --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                                    --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
                                    --blackduck.trust.cert=true \
                                    --logging.level.com.synopsys.integration=DEBUG \
                                    --detect.project.name="CloudBeesInsecureBank-FailBuild" \
                                    --detect.tools="DOCKER,BINARY_SCAN" \
                                    --detect.binary.scan.file.path="/opt/blackduck/shared/target/cloudbees_detect_app.tar" \
                                    --detect.docker.image="cloudbees_detect_app:latest" \
                                    --detect.project.version.name="ALL_${BUILD_TAG}" \
                                    --detect.report.timeout=9000 \
                                    --detect.docker.passthrough.imageinspector.service.url="http://blackduck-imageinspector-alpine.blackduck-imageinspector" \
                                    --detect.docker.passthrough.shared.dir.path.local="/opt/blackduck/shared/" \
                                    --detect.docker.passthrough.shared.dir.path.imageinspector="/opt/blackduck/shared" \
                                    --detect.docker.passthrough.imageinspector.service.start=false \
                                    --detect.policy.check.fail.on.severities' 
                        }
                }
              }

                stage('Black Duck Binary Analysis') {
                    agent { label "python-app" }
                    steps {
                        container('python') {
                          slackSend color: 'good', message: "Performing Black Duck Binary Analysis"
                            sh 'python /opt/blackduck/bdba-pdf.py \
                                --app="/opt/blackduck/shared/target/cloudbees_detect_app.tar" \
                                --protecode-host="protecode-sc.com" \
                                --protecode-username="gautamb@synopsys.com" \
                                --protecode-password="${PROTECODE_SC_PASSWORD}" \
                                --protecode-group="Duck Binaries"'
                            sh 'find . -type f -iname "*/**.pdf" -exec tar -cf synopsys_scan_results.tar "{}" +'
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
          slackSend color: 'good', message: "Publishing Image"
          container('docker-with-detect') {
            //unstash 'detectReport'
            unstash 'bdbaReport'
            sh 'find . -type f -iname "*.pdf" -exec tar -cf synopsys_scan_results.tar "{}" +'
            archiveArtifacts artifacts: '**.tar', fingerprint: true, onlyIfSuccessful: true
            sh 'cat my_password.txt | docker login --username gautambaghel --password ${DOCKER_LOGIN_PASSWORD}'
            sh 'docker tag cloudbees_detect_app:latest gautambaghel/cloudbees_detect_app:latest'
            sh 'docker push gautambaghel/cloudbees_detect_app:latest'
          }
        }
      }

      stage('Deploy + IAST'){
             agent {label 'tomcat-war'}
             steps{
               slackSend color: 'good', message: "Deploying image to Tomcat"
               container('tomcat') {
                 unstash 'warfile'
                 sh "cp target/insecure-bank.war /usr/local/tomcat/webapps/"
                 sh "/usr/local/tomcat/bin/startup.sh"
                 sleep 45
                }

                 slackSend color: 'good', message: 'Initiating IAST with Seeker'
             container('arachni'){
               sh '/sectools/arachni-1.5.1-0.5.12/bin/arachni --checks=csrf http://aaf00f943ce0911e9a6c70217af26a80-860603976.us-east-2.elb.amazonaws.com/insecure-bank/ --report-save-path=Arachni-report.afr'
               sh '/sectools/arachni-1.5.1-0.5.12/bin/arachni_reporter Arachni-report.afr --report=xml:outfile=Arachni-Report.xml'
               sleep 30
               sh "curl -X GET --insecure --header 'accept: */*' --header 'format: JSON' --header 'ver: 1' --header 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJTZWVrZXIiLCJuYW1lIjoiQ2xvdWRCZWVzLVBvbGFyaXNSZXBvcnRzIiwidHlwZSI6ImFwaSIsImlhdCI6MTU3MDYzNzAxMX0.XMgxLw3kbrTdRhkHVa6cg-_jALSNwAhOfuX9D0ltraY' 'http://ec2-18-218-251-228.us-east-2.compute.amazonaws.com:8080/rest/api/latest/vulnerabilities?projectKeys=cloudbeesinsecurebank&format=json' > Seeker-Report.json "
               archiveArtifacts 'Seeker-Report.json,**/Arachni-Report.xml'
               stash includes: '**/Seeker-Report.json', name: 'SeekerReport'
             }
           }
         }


  }
 }
