pipeline {
  agent none
  environment {
      BLACKDUCK_ACCESS_TOKEN  = credentials('jenkins-blackduck-access-token')
  }
  stages {

      stage('Build') {
        agent { label 'maven-app' }
        steps {
          container('maven') {
            sh 'mvn clean package'
            stash includes: 'target/', name: 'builtSources'
          }
        }
      }

      stage('Detect') {
        agent { label 'detect-app' }
        when {
          expression {
            currentBuild.result == null || currentBuild.result == 'SUCCESS'
          }
        }
        steps {
          container('detect') {
            unstash 'builtSources'
            sh 'ls'
            sh '/opt/blackduck/detect.sh \
                --blackduck.url="https://bizdevhub.blackducksoftware.com" \
                --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
                --blackduck.trust.cert=true \
                --detect.project.name="CloudBeesInsecureBank" \
                --detect.tools="SIGNATURE_SCAN" \
                --detect.project.version.name="${BUILD_TAG}" \
                --detect.risk.report.pdf=true \
                --detect.report.timeout=9000'
            sh 'find  . -type f -iname "*.pdf" -exec tar -rvf synopsys_scan_results.tar "{}" +'
            archiveArtifacts artifacts: '**/*.tar', fingerprint: true, onlyIfSuccessful: true
          }
        }
      }

  }
}
