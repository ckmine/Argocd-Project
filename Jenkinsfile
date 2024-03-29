pipeline { 
         agent any
         
           environment {
        jenkins_server_url = "http://192.168.163.120:8080"
        notification_channel = 'devops'
        slack_url = 'https://hooks.slack.com/services/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'
        deploymentName = "web-server"
    containerName = "web-server"
    serviceName = "web-server"
    imageName = "master.mine.com/holder/$JOB_NAME:v1.$BUILD_ID"
     applicationURL="http://192.168.163.120"
    applicationURI="epps-smartERP/" 		   
		   
        
    }
         
    
    tools {
        maven 'maven3'
	//git 'git-tool'
    }
  
    stages { 
        stage('Build Checkout') { 
            steps { 
              checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/ckmine/Argocd-Project.git']]])
         }
        }
        stage('Build Now') { 
            steps { 
              
                  dir("/var/lib/jenkins/workspace/jenkins-with-argocd") {
                    sh 'mvn -version'
                    sh 'mvn clean install'
                      
                    echo "build succses"
                }

       
              }

            }
            
            
            
             
              stage ('Code Quality scan') {
              steps {
        withSonarQubeEnv('sonar') {
          
       sh "mvn clean verify sonar:sonar -Dsonar.projectKey=mine-project -Dsonar.host.url=http://192.168.163.120:9000 -Dsonar.login=sqp_eabeb4419c50b72e518a6d841ef6d6888950d1df"
      
        }
		      timeout(time: 2, unit: 'HOURS') {
           script {
             waitForQualityGate abortPipeline: true
           }
         }
   }
              }
              
              
         stage('Synk-Test') {
      steps {
	      snykSecurity failOnError: false, failOnIssues: false, projectName: 'holder', snykInstallation: 'snyk', snykTokenId: 'snyk'
       // echo 'Testing...'
      //  snykSecurity(
      //    snykInstallation: 'snyk',
      //    snykTokenId: '4ccc3f60-b327-420b-8e0e-7eaf0521560c',
          // place other parameters here
       // )
      }
    }
              
              
              stage ('Vulnerability Scan - Docker ') {
              steps {
                  
                 parallel   (
       "Dependency Scan": {
       	     	sh "mvn dependency-check:check"
		},
	 	  "Trivy Scan":{
	 		    sh "bash trivy-docker-image-scan.sh"
		     	},
		   "OPA Conftest":{
			sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
		    }   	
		             	
   	                      )
                    
              }
               }
              
           
              stage(' Rename and move Build To Perticuler Folder '){
                steps {
                   sh 'mv /var/lib/jenkins/workspace/jenkins-with-argocd/target/jenkins-git-integration.war   /var/lib/jenkins/workspace/jenkins-with-argocd/epps-smartERP.war'
                  sh 'chmod -R 777 /var/lib/jenkins/workspace/jenkins-with-argocd/epps-smartERP.war'
                  
                  sh 'chmod -R 777 /var/lib/jenkins/workspace/jenkins-with-argocd/Dockerfile'
                  sh 'chmod -R 777 /var/lib/jenkins/workspace/jenkins-with-argocd/shell.sh'
                  sh 'chown jenkins:jenkins  /var/lib/jenkins/workspace/jenkins-with-argocd/trivy-docker-image-scan.sh'                
                 
                                     }
                       }
                       
                       stage ("Slack-Notify"){
                         steps {
                            slackSend channel: 'devops', message: 'deployment successfully'
                         }
                       }

    stage ('Regitsry Approve') {
      steps {
      echo "Taking approval from DEV Manager forRegistry Push"
        timeout(time: 7, unit: 'DAYS') {
        input message: 'Do you want to deploy?', submitter: 'admin'
        }
      }
    }

 // Building Docker images
    stage('Building image | Upload to Harbor Repo') {
      steps{
            sh '/var/lib/jenkins/workspace/jenkins-with-argocd/shell.sh'  
    }
      
    }
    
    
	 stage('Vulnerability Scan - Kubernetes') {
       steps {
         parallel(
           "OPA Scan": {
             sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego blue.yml'
         },
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          },
           "Trivy Scan": {
             sh "bash trivy-k8s-scan.sh"
           }
        )
      }
    }
   
/*	    
 stage('Update Manifest for test'){
    steps{
        dir("/var/lib/jenkins/workspace/jenkins-with-argocd/secret"){
           sh  "sed -i 's#replace#${imageName}#g' blue.yml"
            sh "cat blue.yml"
        }
   }
}	
*/
 stage('Build Checkout-argo') { 
            steps { 
             checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'git-new-update', url: 'https://github.com/ckmine/argo-cd-deployment-part.git']])       
	    }
        } 
	    
	
	 stage('commit & push'){
  steps{
	 withCredentials([gitUsernamePassword(credentialsId: 'git-new-update', gitToolName: 'git')]) 
    dir("/var/lib/jenkins/workspace/jenkins-with-argocd"){
        sh  "sed -i 's#replace#${imageName}#g' blue.yaml"         
	sh "cat blue.yaml"   
        sh 'git add blue.yaml'
        sh 'git commit -am "update ${imageName}"'
        sh 'remote -v'
        sh 'git push https://github.com/ckmine/argo-cd-deployment-part.git HEAD:main' 
    }
    }
  }
	    
	    
	    
    /*
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
	    
    */
	    
	   stage('Integration Tests - DEV') {
         steps {
         script {
          try {
            withKubeConfig([credentialsId: 'kubeconfig']) {
               sh "bash integration-test.sh"
             }
            } catch (e) {
             withKubeConfig([credentialsId: 'kubeconfig']) {
               sh "kubectl -n test rollout undo deploy ${deploymentName}"
             }
             throw e
           }
         }
       }
     }  
	    
	 stage('OWASP ZAP - DAST') {
       steps {
         withKubeConfig([credentialsId: 'kubeconfig']) {
           sh 'bash zap.sh'
         }
       }
     }  
	    
	    stage('Prompte to PROD?') {
       steps {
         timeout(time: 2, unit: 'DAYS') {
           input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
         }
       }
     }

     stage('K8S CIS Benchmark') {
       steps {
         script {

           parallel(
            "Master": {
               sh "bash cis-master.sh"
             },
             "Etcd": {
               sh "bash cis-etcd.sh"
             },
             "Kubelet": {
               sh "bash cis-kubelet.sh"
             }
           )

         }
       }
     }
	    
	 stage('Update Manifest for Prod'){
    steps{
        dir("/var/lib/jenkins/workspace/jenkins-with-argocd/secret"){
            sh  "sed -i 's#replace#${imageName}#g' k8s_PROD-deployment_service.yaml"
            sh "cat k8s_PROD-deployment_service.yaml"
        }
    }
}	     
	    
	    
	    
	    stage('commit & push for Prod'){
  steps{
	 withCredentials([gitUsernamePassword(credentialsId: 'gitcred', gitToolName: 'git')]) 
    dir("/var/lib/jenkins/workspace/jenkins-with-argocd"){
        sh "git config --global user.email 'ck769184@gmail.com'"
        sh 'git remote set-url origin https://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@github.com/ckmine/Argocd-Project.git'        
        sh 'git add secret'
        sh 'git commit -am "update ${imageName}"'
        sh 'git push origin HEAD:main' 
    }
    }
  }
	    
	  /*  
	    
	  stage('K8S Deployment - PROD') {
       steps {
         parallel(
           "Deployment": {
             withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "sed -i 's#replace#${imageName}#g' k8s_PROD-deployment_service.yaml"
               sh "kubectl -n prod apply -f k8s_PROD-deployment_service.yaml"
             }
           },
           "Rollout Status": {
             withKubeConfig([credentialsId: 'kubeconfig']) {
               sh "bash k8s-PROD-deployment-rollout-status.sh"
             }
           }
         )
       }
     }  
	*/
	    
	    
	    
	    
	    
	    
	    
	    stage('Integration Tests - PROD') {
       steps {
         script {
          try {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash integration-test-PROD.sh"
             }
           } catch (e) {
             withKubeConfig([credentialsId: 'kubeconfig']) {
               sh "kubectl -n prod rollout undo deploy ${deploymentName}"
             }
             throw e
           }
         }
       }
     }  
	    
     
}

			 post{
                      always{
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
              publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report', useWrapperFileDirectly: true])
       }
   }

    }
