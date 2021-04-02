pipeline {
    agent any
    environment {
	   // which region do you want to deploy in aws
    awsRegion = "us-east-1"
	  awsAccoundId = "558768252711"
    awsProdAccoundId = "558768252711"
	  notification_mail = "babu@ensarsolutions.com"   
	   // add the assume role which account you want to deploy 
	   awsAssumeRole = "${env.BRANCH_NAME == "develop" ? "${awsAssumeRole_dev}" : "${env.BRANCH_NAME}" == "release" ? "${awsAssumeRole_stg}" : " ${env.BRANCH_NAME}" == "hotfix" ? "${awsAssumeRole_stg}" : "${awsAssumeRole_prod}"}"
	   awsAssumeRole_stg = "arn:aws:iam::558768252711:role/Sample-Jenkins-Role-virgnia"
	   awsAssumeRole_dev = "arn:aws:iam::558768252711:role/Sample-Jenkins-Role-virgnia"
		 awsAssumeRole_prod = "arn:aws:iam::558768252711:role/Sample-Jenkins-Role-virgnia"


	   // service name 
	   SERVICE = "api"
	   
	   //cloudformation file name
	   cfname = "2-api.yaml"
                 
	   // cloudformation stack names
	   stack_name = "${env.BRANCH_NAME == "develop" ? "${stack_dev}" : "${env.BRANCH_NAME}" == "release" ? "${stack_stg}" : " ${env.BRANCH_NAME}" == "hotfix" ? "${stack_stg}" : "${stack_stg}"}"
	   stack_dev = "test-dev"
	   stack_stg = "test-stg"
     S3_BUCKET_NAME = "tt-env-files"
	   


	   // environment deployment
     ENVIRONMENT = "${env.BRANCH_NAME == "develop" ? "dev" : "${env.BRANCH_NAME}" == "release" ? "stg" : " ${env.BRANCH_NAME}" == "hotfix" ? "prd" : "stg"}"
	   // provide the ecr repository details
	   ECR_REPOSITORY = "${env.BRANCH_NAME == "develop" ? "${ECR_REPOSITORY_DEV}" : "${env.BRANCH_NAME}" == "release" ? "${ECR_REPOSITORY_STG}" : " ${env.BRANCH_NAME}" == "hotfix" ? "${ECR_REPOSITORY_STG}" : "${PRODECR_REPO}"}"
	   ECR_REPOSITORY_DEV = "558768252711.dkr.ecr.us-east-1.amazonaws.com/tt-dev/api"
	   ECR_REPOSITORY_STG = "558768252711.dkr.ecr.us-east-1.amazonaws.com/tt-stg/api"
	   PRODECR_REPO = "558768252711.dkr.ecr.us-east-1.amazonaws.com/tt-prod/api"
	   ECR_ARTIFACTS = "tt-prd/api"
	   ARTIFACTS_LOCATION = "https://${awsRegion}.console.aws.amazon.com/ecr/repositories/${ECR_ARTIFACTS}/?region=${awsRegion}"
	   GIT_COMMIT_AUTHOR = sh (script: "git log -n 1 --pretty=format:'%ae'", returnStdout: true)
    scmRevisionNumber = sh (script: "git log --format='%H' -n 1", returnStdout: true )
	   

	   
  }
  triggers {
    pollSCM ('* * * * *')
	// cron ('0 0 * * 1-5')
  }
  options {
      disableConcurrentBuilds()
      buildDiscarder(logRotator(numToKeepStr: '5', artifactDaysToKeepStr: '3', artifactNumToKeepStr: '1'))
      timeout(time: 150, unit: 'MINUTES') 
   }
   stages {
     stage('SonarCode Quality Gate Statuc Check') {
        steps {
          script {
           def scannerHome = tool 'SoncarScanner';
	withSonarQubeEnv(credentialsId: 'SonarQube-key') {
	  sh "npm install"
	  sh "${tool("SoncarScanner")}/bin/sonar-scanner -Dsonar.sources=. -Dsonar.projectKey=tt-api"
            }
             timeout(time: 1, unit: 'HOURS') {
             def qg = waitForQualityGate()
             if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
               }
          }
        }
     }
     }
     stage('Last changes html diff') {
	steps {
               script {
               def publisher = LastChanges.getLastChangesPublisher "PREVIOUS_REVISION", "SIDE", "LINE", true, true, "", "", "", "", ""
               publisher.publishLastChanges()
               def htmlDiff = publisher.getHtmlDiff()
               writeFile file: 'build-diff.html', text: htmlDiff	 
		 }
	}
          }		 
     stage('Build App Code'){
       steps {
           script {
	// echo "====================== Building Docker Image ======================"
	// dockerImage = docker.build "${ECR_REPOSITORY}" + ":${BUILD_NUMBER}"
  withAWS(region:"${awsRegion}", role:"${awsAssumeRole}") {
     sh '''
     aws s3 cp s3://${S3_BUCKET_NAME}/api/${ENVIRONMENT}.env ./
     mv ${ENVIRONMENT}.env .env'''
     	echo "====================== Building Docker Image ======================"
	    dockerImage = docker.build "${ECR_REPOSITORY}" + ":${ENVIRONMENT}-${BUILD_NUMBER}"
	   echo "====================== Image pushing to ecr repo ================="
	   sh '(eval \$(aws ecr get-login  --no-include-email --region "${awsRegion}"))'
	   dockerImage.push()
              }
           }
       } 
    }
   stage('Deploy Dev Env'){
     when {
	branch 'develop'
	 }
      steps {
          script {
                println("=================== deploying Dev environment =============")
	  withAWS(region:"${awsRegion}", role:"${awsAssumeRole}") {
	    cfnCreateChangeSet(changeSet: "${stack_name}", stack:"${stack_name}", file:"${cfname}", params:['ImageUrl': "${ECR_REPOSITORY}:${BUILD_NUMBER}",  'Environment':"${ENVIRONMENT}", ], timeoutInMinutes:30) 
                  cfnExecuteChangeSet(changeSet: "${stack_name}", stack:"${stack_name}", timeoutInMinutes:30)
		}
	   echo "================ Sending email notifications for Dev...... ================="
	  currentBuild.result = "SUCCESS"
          }
             notifyEmaildev()
       }
     }	 

     stage('Deploy Stg Env') {
	when { 
    anyOf 
      {  
        branch 'release'; branch 'hotfix' 
        }
   }
	steps {
	   script {
	     println("============== deploying stg environment ================")
	       withAWS(region:"${awsRegion}", role:"${awsAssumeRole}") {
	         cfnCreateChangeSet(changeSet: "${stack_name}", stack:"${stack_name}", file:"${cfname}", params:['ImageUrl': "${ECR_REPOSITORY}:${BUILD_NUMBER}", 'Environment':"${ENVIRONMENT}", ], timeoutInMinutes:30) 
                       cfnExecuteChangeSet(changeSet: "${stack_name}", stack:"${stack_name}", timeoutInMinutes:30)
	         }
	     echo "==================== Sending email notifications for stg.... ==========="
	     currentBuild.result = "SUCCESS"			   
	      }
	    notifyEmailstg()
	     }
	   }
	   
     stage('QA Team Approval') {
	when {
	   expression {
   	        return env.BRANCH_NAME == 'release'
	             }
		 }
         steps {
             script {
	   REQUESTED_ACTION = input message: "Please provide qa sign off for prod", ok: "should we continue?", 
	   parameters: [choice(name: 'APPROVAL_ACTION', choices: ['YES', 'NO'], description: 'please enter aproval')]
             }
    timeout(time:7, unit:'DAYS') {
	     echo "User Input: ${REQUESTED_ACTION}"
    }

         }
      }
    stage('Sending Email') {
        when {
	expression {
   	   return env.BRANCH_NAME == 'release'
		}
	       }
	  steps {
	      script {
		     echo "==============Sending  Email================="
		  }
		   emailext body: """ Hi Devops, \n
		     <p>QA sign off: ${REQUESTED_ACTION}.</p>
		     <p>Environment: ${ENVIRONMENT} </p>
			 <BR/><BR/><BR/>
			 <p>Thank you,
             <br>Devops Team</p>""",      
             subject: "QA sign off ", 
             to: "${notification_mail}"
	   }
	 }	  
      stage('Docker Image Build for Production Perpose') {
	     when {
		expression {
   		   return env.BRANCH_NAME == 'release' && REQUESTED_ACTION == "YES"
		   }
		 }
	     steps {
                     script {
	            echo "====================== Building Docker Image for Production ======================"
	            dockerImage = docker.build "${PRODECR_REPO}" + ":${BUILD_NUMBER}"
                          withAWS(region:"${awsRegion}", role:"${awsAssumeRole_prod}") {
	            echo "====================== Image pushing to ecr repo ================="
	            sh '(eval \$(aws ecr get-login  --no-include-email --region "${awsRegion}"))'
	            dockerImage.push()
                        }
                        }
	          }
                   }
      stage('Sending Email..') {
	    when {
		expression {
   		  return env.BRANCH_NAME == 'release' && REQUESTED_ACTION == 'YES'
			 }
		}
		steps{
		   println("Sending email...")
		   script {
		       VERSION = "${BUILD_NUMBER}"
		   }
		   emailext body: """ Hi Devops, \n
		     <p> ${VERSION} has been built and is available for use in production API APP.</p>
		     <p>It is available at aws ecr repository. Version ID: <a href="${ARTIFACTS_LOCATION}">${VERSION}.zip</a></p>
			 <BR/><BR/><BR/>
			 <p>Thank you,
             <br>Devops Team</p>""",      
             subject: "${SERVICE} service Release version: ${VERSION}", 
             to: "${notification_mail}"
		}    
	  }
  
     stage('Delete workspace') {
	    steps {
	script {
	echo "==============Deleting workspace and docker Image================="
             sh '''
                docker system prune -af
             '''
             echo "Docker image has been deleted in jenkis machine"
		   }
		  //  cleanWs()
		   echo "Deleted workspce"
		}
      }	 
    } 
   post {
      success {
      echo 'Successful build occured'
      script {
          currentBuild.result = "SUCCESS"
      }
    }

    failure {
      echo 'Failure occured sending notification'
      script {
        currentBuild.result = "FAILURE"
      }
      failure()
    }
   }
  }
def notifyEmaildev () {
       emailext attachLog: false,
           body: '''${JELLY_SCRIPT, template="api-dev-approval"}''',
           attachmentsPattern: '**/*build-diff.html',
           subject: "Status: ${currentBuild.result?:'SUCCESS'} - Job \'${env.JOB_NAME}:${env.BUILD_NUMBER}\'",
           to: "${GIT_COMMIT_AUTHOR},"+ '$DEFAULT_RECIPIENTS'
}
def notifyEmailstg () {
      emailext attachLog: false,
           body: '''${JELLY_SCRIPT, template="api-stg-approval"}''',
           attachmentsPattern: '**/*build-diff.html',
           subject: "Status: ${currentBuild.result?:'SUCCESS'} - Job \'${env.JOB_NAME}:${env.BUILD_NUMBER}\'",
           to: "${GIT_COMMIT_AUTHOR},"+ '$DEFAULT_RECIPIENTS'

}
def notifyEmailhotfix () {
      emailext attachLog: false,
           body: '''${JELLY_SCRIPT, template="api-hotfix-approval"}''',
           attachmentsPattern: '**/*build-diff.html',
           subject: "Status: ${currentBuild.result?:'SUCCESS'} - Job \'${env.JOB_NAME}:${env.BUILD_NUMBER}\'",
           to: "${GIT_COMMIT_AUTHOR},"+ '$DEFAULT_RECIPIENTS'

}


def failure() {
      emailext attachLog: false,
           body: '''${JELLY_SCRIPT, template="api-error"}''',
           attachmentsPattern: '**/*build-diff.html',
           subject: "Status: ${currentBuild.result?:'FAILURE'} - Job \'${env.JOB_NAME}:${env.BUILD_NUMBER}\'",
           to: "${GIT_COMMIT_AUTHOR},"+ '$DEFAULT_RECIPIENTS'
  }
