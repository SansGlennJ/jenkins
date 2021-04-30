pipeline {
  agent any
  environment {
    no_proxy = "localhost"
  }
  parameters {    
    string(name: 'CONTAINER', defaultValue: "hub.docker.com/r/jenkins/jenkins:lts", description: 'Full path to the container to scan.'
    booleanParam(name: 'DEDUPE_ENABLED', defaultValue: true)
  }
  options {
    buildDiscarder(
      logRotator(numToKeepStr: "5", artifactNumToKeepStr: "5")
     )
  }
  stages {       
    stage('Scan Containers') {
      parallel {
        stage('Dive') {
          agent { label 'dso_eco_linux'} 
          steps {
            sh "docker pull ${params.CONTAINER} > /dev/null"
            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v ${env.WORKSPACE}:/opt/data wagoodman/dive --ci --json /opt/data/dive.json ${params.CONTAINER}"
            archiveArtifacts artifacts: 'dive.json'
          }
          post { 
            always { 
                sh 'echo Cleaning Workspace'
                cleanWs()
            }
          }
        }
        stage('Dockle') {
          agent { label 'dso_eco_linux' }   
          steps {
            script {
              sh "docker pull ${params.CONTAINER} > /dev/null"
              sh "docker pull hub.docker.com/r/goodwithtech/dockle"
              try {
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker-ng-proxy.ms.northgrum.com/goodwithtech/dockle:v0.3.1-1.0.0 -f json --timeout 5m --exit-code 0 ${params.CONTAINER} > dockle.json"
              } catch (err) {
                echo "[ERROR] Error running the Dockle scan."
                sh "cat dockle.json"
              } finally {
                archiveArtifacts artifacts: 'dockle.json'
              }
            }
          }
          post { 
            always { 
                sh 'echo Cleaning Workspace'
                cleanWs()
            }
          }
        }
        stage('Trivy') { 
          agent { label 'dso_eco_linux' }     
          steps {                            
            sh 'mkdir -p trivy'
            sh 'docker pull aquasec/trivy:latest'
            sh 'docker run -e https_proxy=$https_proxy --rm -v /var/run/docker.sock:/var/run/docker.sock -v "$PWD/trivy":/root/.cache/ aquasec/trivy --download-db-only'         
            sh "docker pull ${params.CONTAINER} > /dev/null"
            sh "docker run -e https_proxy=$https_proxy --rm -v /var/run/docker.sock:/var/run/docker.sock -v \"$PWD/trivy\":/root/.cache/ aquasec/trivy -f json --timeout 5m -q ${params.CONTAINER} > trivy.json"
            archiveArtifacts artifacts: 'trivy.json'
            stash name: "trivy", includes: "trivy.json"
          }
          post { 
            always { 
                sh 'echo Cleaning Workspace'
                cleanWs()
            }
          }
        }
            }
          }
        }
      }
          post { 
            always { 
                sh 'echo Cleaning Workspace'
                cleanWs()
            }  
          }          
        }  
      }  
    }  
    stage('Generate Report') {
      agent { label 'dso_eco_linux' } 
      steps {
        sh 'docker pull hub.docker.com/csf/report:1.0.0-9'
        unstash "trivy"
        withDockerContainer(image: 'hub.docker.com/csf/report:1.0.0-9') {
          script {
            if (params.DEDUPE_ENABLED) {
              sh '/app/csf-report --outputName=report --deduplicate=true --trivy=trivy.json --dockle=dockle.json --dive=dive.json' 
            } else {
              sh '/app/csf-report --outputName=report --deduplicate=false --trivy=trivy.json --dockle=dockle.json --dive=dive.json'
            }
          }
        }
        archiveArtifacts artifacts: 'report.json'
        archiveArtifacts artifacts: 'report.csv'
        archiveArtifacts artifacts: 'report.html'
      }
      post { 
        always { 
          sh 'echo Cleaning Workspace'
          cleanWs()
          rtp stableText: '<script>document.getElementById("description").innerHTML = "<h1>Quick Start -> Click Build with Parameters option on left hand menu</h1><h1</a></h1>"</script>', unstableAsStable: true, failedAsStable: true, parserName: 'HTML', abortedAsStable: true
        }
      }
    }
  }    
}
