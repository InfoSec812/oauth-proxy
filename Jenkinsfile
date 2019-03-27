pipeline {
  // Use Jenkins Maven slave
  // Jenkins will dynamically provision this as OpenShift Pod
  // All the stages and steps of this Pipeline will be executed on this Pod
  // After Pipeline completes the Pod is killed so every run will have clean
  // workspace
  agent {
    label 'jenkins-slave-golang'
  }

  // Pipeline Stages start here
  // Requeres at least one stage
  stages {

    // Clone git repo
    stage('Clone'){
        steps {
            dir('src/github.com/openshift/oauth-proxy') {
                git branch: 'Issue-107_-_Add_option_to_pass_Authorization_Bearer_to_upstream', url: 'https://github.com/bvkin/oauth-proxy.git'
            }
        }
    }

    // Run Go test  
    stage('Test'){
      steps {
        dir('src/github.com/openshift/oauth-proxy') {
          sh '''
          pwd
          export GOPATH=$(echo $PWD | sed 's@/src/.*@/@g')
          mkdir reports
          go test -coverprofile=reports/coverage.out 
          go tool cover -html=reports/coverage.out -o reports/coverage.html
          '''
          publishHTML(target: [
            reportDir:             'reports',
            reportFiles:           'coverage.html',
            reportName:            'Code Coverage report', 
            keepAll:               true,
            alwaysLinkToLastBuild: true,
            allowMissing:          true
          ]) 
          }
        }
      }


    // Run Go build, skipping tests
    stage('Build'){
      steps {
        dir('src/github.com/openshift/oauth-proxy') {
          sh '''
          pwd
          export GOPATH=$(echo $PWD | sed 's@/src/.*@/@g')
          go build 
          '''
          }
        }
      }

      stage("Build Container Image") {
          steps {
              
              dir('src/github.com/openshift/oauth-proxy') {
                 script {
                    openshift.withCluster() {
                            openshift.withProject("labs-ci-cd") {
                                timeout(5) {
                                  def bc = openshift.newBuild('--name=oauth-proxy', '--binary=true', '--to=oauth-proxy')
                                  bc.startBuild("--from-file=./oauth-proxy", "--wait")
                                }
                            }
                    }
                 }
          }  }
      }
      
      stage("Promote Container Image") {
          steps {
              script {
                    openshift.withCluster() {
                        openshift.tag("itcm-ngm-operator:latest", "labs-test/itcm-ngm-operator:0.0.1.${BUILD_ID}")
                    }
              }
          }
      }
      
      stage("Deploy the Operator To Test") {
         steps {
               script {
            openshift.withCluster() {
                  openshift.withProject("labs-test") {
          def deploy = openshift.selector('deploy/itcm-ngm-operator').object()
          deploy.spec.template.spec.containers[0].image ="docker-registry.default.svc:5000/labs-test/itcm-ngm-operator:0.0.1.${BUILD_ID}"
          openshift.apply(deploy)
                  }
            }
               }
         }         
         }
         
         
       stage("Promote To Dev") {
          options {
            timeout(time: 1, unit: 'HOURS')
          }
          steps {
              script {
                input message: 'Deploy to Dev?'
              }
              script {
                    openshift.withCluster() {
                        openshift.tag("labs-test/itcm-ngm-operator:0.0.1.${BUILD_ID}", "labs-dev/itcm-ngm-operator:0.0.1.${BUILD_ID}")
                    }
              }
          }
      }
      
      stage("Deploy the Operator To Dev") {
         steps {
               script {
            openshift.withCluster() {
                  openshift.withProject("labs-dev") {
          def deploy = openshift.selector('deploy/itcm-ngm-operator').object()
          deploy.spec.template.spec.containers[0].image ="docker-registry.default.svc:5000/labs-dev/itcm-ngm-operator:0.0.1.${BUILD_ID}"
          openshift.apply(deploy)
                  }
            }
               }
         }         
         }
  }
}

