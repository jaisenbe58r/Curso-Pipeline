pipeline {
  agent any
  stages {
    stage('Code pull') {
      steps {
        checkout scm
      }
    }

    stage('Build environment') {
      steps {
        echo 'Building virtualenv'
        sh ''' conda create --yes -n ${BUILD_TAG} python
                        source activate ${BUILD_TAG}
                        pip install -r requirements/dev.txt
                    '''
      }
    }

    stage('Static code metrics') {
      steps {
        echo 'Raw metrics'
        sh ''' source activate ${BUILD_TAG}
                        radon raw --json irisvmpy > raw_report.json
                        radon cc --json irisvmpy > cc_report.json
                        radon mi --json irisvmpy > mi_report.json
                    '''
        echo 'Test coverage'
        sh ''' source activate ${BUILD_TAG}
                        coverage run irisvmpy/iris.py 1 1 2 3
                        python -m coverage xml -o reports/coverage.xml
                    '''
        echo 'Style check'
        sh ''' source activate ${BUILD_TAG}
                        pylint irisvmpy || true
                    '''
      }
    }

    stage('Unit tests') {
      post {
        always {
          junit(allowEmptyResults: true, testResults: 'reports/unit_tests.xml')
        }

      }
      steps {
        sh ''' source activate ${BUILD_TAG}
                        python -m pytest --verbose --junit-xml reports/unit_tests.xml
                    '''
      }
    }

    stage('Acceptance tests') {
      post {
        always {
          cucumber(buildStatus: 'SUCCESS', fileIncludePattern: '**/*.json', jsonReportDirectory: './reports/', sortingMethod: 'ALPHABETICAL')
        }

      }
      steps {
        sh ''' source activate ${BUILD_TAG}
                        behave -f=formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true
                    '''
      }
    }

    stage('Build package') {
      when {
        expression {
          currentBuild.result == null || currentBuild.result == 'SUCCESS'
        }

      }
      post {
        always {
          archiveArtifacts(allowEmptyArchive: true, artifacts: 'dist/*whl', fingerprint: true)
        }

      }
      steps {
        sh ''' source activate ${BUILD_TAG}
                        python setup.py bdist_wheel
                    '''
      }
    }

  }
  environment {
    PATH = "/var/lib/jenkins/miniconda3/bin:$PATH"
  }
  post {
    always {
      sh 'conda remove --yes -n ${BUILD_TAG} --all'
    }

    failure {
      emailext(subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'", body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                               <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""", recipientProviders: [[$class: 'DevelopersRecipientProvider']])
    }

  }
  options {
    skipDefaultCheckout(true)
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
  }
  triggers {
    pollSCM('*/5 * * * 1-5')
  }
}