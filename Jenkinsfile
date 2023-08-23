pipeline {
	agent any
	stages {
	//---------------------------------------------------	
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar'  //test
            }
        }   
//---------------------------------------------------
  stage('UNIT test & jacoco ') {
      steps {
        sh "mvn test"
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
        }
      }

    }
//---------------------------------------------------
	}
}
