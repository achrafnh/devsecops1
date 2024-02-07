pipeline {
  agent any

	tools {
        // Specify the JDK configured in the Global Tool Configuration
        jdk 'Java 11'
    }
//--------------------------
  stages {
    stage('Build Artifact') {
      steps {
        sh "mvn clean package -DskipTests=true"
        archive 'target/*.jar'
      }
    }


    //--------------------------
    stage('UNIT test & jacoco ') {
      steps {
        sh "mvn test"
      }


    }
//--------------------------
    stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }

    }


//--------------------------

     stage('SonarQube - SAST') {
       steps {
         withSonarQubeEnv('SonarQubeConfig') {
           sh "mvn sonar:sonar \
  -Dsonar.projectKey=project-achraf \
  -Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9999"
         }
         timeout(time: 2, unit: 'MINUTES') {
           script {
             waitForQualityGate abortPipeline: true
           }
         }
       }
     }
//--------------------------
	 stage('Vulnerability Scan - Docker Trivy') {
       steps {
	        withCredentials([string(credentialsId: 'trivy_token', variable: 'TOKEN')]) {
			 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
	 sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh"      
	 sh "sudo bash trivy-image-scan.sh"
	       }
		}
       }
     }
//--------------------------

       //     stage('old working - SonarQube - SAST') {
          
      //     steps {
	//	   catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
      //   withSonarQubeEnv('SonarQube') {
          //  sh "mvn sonar:sonar \
 // -Dsonar.projectKey=project-achraf \
 // -Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9999 \
 // -Dsonar.login=220bc162accb9564166b764d5343595dc0c3f5d8"
      //   }
	//	   }
     //  }
     //}

	//--------------------------  

stage('Vulnerability Scan - Docker') {
   steps {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
     		sh "mvn dependency-check:check"
	    }
		}
	
 }



 //--------------------------

    
//          stage('SonarQube - SAST') {
//          
//           steps {
//         withSonarQubeEnv('SonarQube') {
//
//  withCredentials([string(credentialsId: 'token-sonar', variable: 'TOKEN_SONAR')]) {
//            sh "mvn clean verify sonar:sonar -Dsonar.projectKey=myprojecttp -Dsonar.projectName='myprojecttp'-Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9112 -Dsonar.token=sqp_c31c51dd109a4d1127e014a427db98873cb01af6"
//         }
// }
//      
//       }
//          
// 
//     }
// //--------------------------
//	
//	  
// stage("SonarQube - qualiyGate Status") {
//            steps {
//	catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//              timeout(time: 3, unit: 'MINUTES') {
//                waitForQualityGate abortPipeline: true
//	      }
//              }
//            }
//          }
//
//	  //--------------------------
//    stage('Vulnerability Scan - Docker') {
//   steps {
//	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//     		sh "mvn dependency-check:check"
//	    }
//		}
//		post { 
//      always { 
//				dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
//				}
//		}
// }

	  
//--------------------------
    stage('Docker Build and Push') {
      steps {
        withCredentials([string(credentialsId: 'docker-hub-password', variable: 'DOCKER_HUB_PASSWORD')]) {
          sh 'sudo docker login -u hrefnhaila -p $DOCKER_HUB_PASSWORD'
          sh 'printenv'
          sh 'sudo docker build -t hrefnhaila/devops-app:""$GIT_COMMIT"" .'
          sh 'sudo docker push hrefnhaila/devops-app:""$GIT_COMMIT""'
        }

      }
    }

    //--------------------------
    stage('Deployment Kubernetes  ') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
               sh "sed -i 's#replace#hrefnhaila/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
               sh "kubectl apply -f k8s_deployment_service.yaml"
             }
      }

    }
    //--------------------------

    stage('Vulnerability Scan - Kubernetes') {
       steps {
         parallel(
           "OPA Scan": {
             sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
           },
           "Kubesec Scan": {
             sh "sudo bash kubesec-scan.sh"
           },
           "Trivy Scan": {
             sh "sudo bash trivy-k8s-scan.sh"
           }
         )
       }
     }


      stage('OWASP ZAP - DAST') {
       steps {
         withKubeConfig([credentialsId: 'kubeconfig']) {
           sh 'sudo bash zap.sh'
         }
       }
     }

	  
  }


	      post {
		        always {
		          junit 'target/surefire-reports/*.xml'
		          jacoco execPattern: 'target/jacoco.exec'
			  dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
  			  pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
		        }
			    success {
		        	
		        }
		
			    failure {
			   
			    }
	      }



}
