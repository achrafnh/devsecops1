
pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "hrefnhaila/devops-app:${GIT_COMMIT}"
    applicationURL="http://mytpm.eastus.cloudapp.azure.com"
    applicationURI="/increment/99"
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

stage('Vulnerability Scan owasp - dependency-check') {
   steps {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
     		sh "mvn dependency-check:check"
	    }
		}
	
 }




	  
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

    //--------------------------
    //stage('Deployment Kubernetes  ') {
     // steps {
      //  withKubeConfig([credentialsId: 'kubeconfig']) {
       //        sh "sed -i 's#replace#hrefnhaila/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
        //       sh "kubectl apply -f k8s_deployment_service.yaml"
         //    }
     // }

    //}


         stage('K8S Deployment - DEV') {
           steps {
             parallel(
               "Deployment": {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "bash k8s-deployment.sh"
                 }
               },
               "Rollout Status": {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "bash k8s-deployment-rollout-status.sh"
                 }
               }
             )
           }
         }

    //--------------------------

        stage('Integration Tests - DEV') {
           steps {
             script {
               try {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "bash integration-test.sh"
                 }
               } catch (e) {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "kubectl -n default rollout undo deploy ${deploymentName}"
                 }
                 throw e
               }
             }
           }
         }
 //--------------------------



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
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP HTML Report', reportTitles: 'OWASP HTML Report', useWrapperFileDirectly: true])
                    sendNotification currentBuild.result
		        }
			  //  success {
		        	
		        //}
		
			  //  failure {
			   
			    //}
	      }



}
