jenkins {
  agent {label 'jenkins-agent'}

  tools {
      jdk 'java17'
      maven 'Maven3'
  }

  stages {
     stage ('Clean WorkSpace'){
      steps 
      {
        cleanWs()
      }
  }

    stage ('Checkout From SCM'){
      steps {
        git branch: 'main', credentialsId: 'GitHub', usl: 'https://gitHub.com/https://github.com/SROR7/register-app'
      }
  }

    stage ('Build Application') {
      steps {
        sh 'mvn clean package'
      }
  }

    stage ('Test Application'){
      steps{
        sh 'mvn test'
      }
  }
}
