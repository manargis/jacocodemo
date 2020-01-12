pipeline {
  agent {
      label 'maven'
  }
  stages {
    stage('Build App') {
      steps {
        sh "mvn install"
      }
    }
    stage('Code Analysis') {
      steps {
        script {
          sh "mvn sonar:sonar -Dsonar.host.url=http://sonarqube-myproject-manargis.osp-apps.k4it.xyz -Dsonar.login=admin -Dsonar.password=admin"
         }
      }
    }    
    stage('Create Image Builder') {
      when {
        expression {
          openshift.withCluster() {
            return !openshift.selector("bc", "springbootapp").exists();
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.newBuild("--name=springbootapp", "--image-stream=openjdk18-openshift:1.1", "--binary")
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.selector("bc", "springbootapp").startBuild("--from-file=target/spring-example-0.0.1-SNAPSHOT.jar", "--wait")
          }
        }
      }
    }
    stage('Promote to UAT') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("springbootapp:latest", "springbootapp:uat")
          }
        }
      }
    }
    stage('Create UAT') {
      when {
        expression {
          openshift.withCluster() {
            return !openshift.selector('dc', 'springbootapp-uat').exists()
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.newApp("springbootapp:latest", "--name=springbootapp-uat").narrow('svc').expose()
          }
        }
      }
    }
    stage('Promote PROD') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("springbootapp:uat", "springbootapp:prod")
          }
        }
      }
    }
    stage('Create PROD') {
      when {
        expression {
          openshift.withCluster() {
            return !openshift.selector('dc', 'springbootapp-prod').exists()
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.newApp("springbootapp:prod", "--name=springbootapp-prod").narrow('svc').expose()
          }
        }
      }  
    }  
    stage('Scan Project') {
		agent { label 'sonar' }
        steps {
            sh "/sonarqube-scanner/bin/sonar-scanner -Dsonar.host.url=http://sonarqube-myproject-manargis.osp-apps.k4it.xyz -Dsonar.login=f91bc8237ef41633899676913e3fc61d481540b5"
        }
      }
    stage('Gated promotion to staging project') {
        agent { label 'base' }
        steps {
          timeout(time:60, unit:'MINUTES') {
            input message: "Would you like to promote the image to the staging project?", ok: "Promote"
          }

        script {
          openshift.withCluster() {
            openshift.tag("project-name/project-name:latest", "project-name/project-name:stage")
			}
		  }
		}
	}
 }
}
