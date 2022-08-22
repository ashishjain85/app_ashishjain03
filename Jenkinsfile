pipeline {
    agent any
		
	environment {
		scannerHome = tool name: 'sonar_scanner_dotnet'
		registry = 'ashishjain85/nagp-devops-us-dotnet'
		username = 'ashishjain03'
        appName = 'nagp-devops-us-dotnet'
   	}	
   
	options {
        //Prepend all console output generated during stages with the time at which the line was emitted.
		timestamps()
		
		//Set a timeout period for the Pipeline run, after which Jenkins should abort the Pipeline
		timeout(time: 1, unit: 'HOURS') 
		
		buildDiscarder(logRotator(
			// number of build logs to keep
            numToKeepStr:'3',
            // history to keep in days
            daysToKeepStr: '15'
        ))
    }
    
    stages {
        
    	stage ("nuget restore") {
            steps {
		    
                //Initial message
                echo "Deployment pipeline started for - ${BRANCH_NAME} branch"

                echo "Nuget Restore step"
                bat "dotnet restore"
            }
        }
		
		stage('Start sonarqube analysis'){
            when {
                branch "master"
            }

            steps {
				  echo "Start sonarqube analysis step"
                  withSonarQubeEnv('Test_Sonar') {
                   bat "dotnet ${scannerHome}\\SonarScanner.MSBuild.dll begin /k:sonar-${userName} /n:sonar-${userName} /v:1.0"
                  }
            }
        }

        stage('Code build') {
            steps {
				  //Cleans the output of a project
				  echo "Clean Previous Build"
                  bat "dotnet clean"
				  
				  //Builds the project and all of its dependencies
                  echo "Code Build"
                  bat 'dotnet build -c Release -o "nagp-devops-us/app/build"'	
            }
        }

        stage('Test case execution') {
            steps {				  
                  echo "Test case execution"
                  bat 'dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover -l:trx;LogFileName=nagp-devops-us.xml'	      
            }
        }

		stage('Stop sonarqube analysis'){
             when {
                branch "master"
            }
            
			steps {
				   echo "Stop sonarqube analysis"
                   withSonarQubeEnv('Test_Sonar') {
                   bat "dotnet ${scannerHome}\\SonarScanner.MSBuild.dll end"
                   }
            }
        }

        stage ("Release artifact") {
            when {
                branch "develop"
            }

            steps {
                echo "Release artifact step"
                bat "dotnet publish -c Release -o ${appName}/app/${userName}"
            }
        }

        stage ("Docker Image") {
            steps {
                //For master branch, publish before creating docker image
                script {
                    if (BRANCH_NAME == "master") {
                        bat "dotnet publish -c Release -o ${appName}/app/${userName}"
                    }
                }
                echo "Docker Image step"
                bat "docker build -t i-${userName}-${BRANCH_NAME}:${BUILD_NUMBER} --no-cache -f Dockerfile ."
                bat "docker build -t i-${userName}-${BRANCH_NAME}:latest --no-cache -f Dockerfile ."
            }
        }

        stage ("Containers") {
            failFast true
            parallel {
                stage ("PrecontainerCheck") {
                    steps {
                        echo "PrecontainerCheck step"
                        script {
						
							if (env.BRANCH_NAME == 'master') {
                                env.port = 7200
                            } else {
                                env.port = 7300
                            }
						
							env.containerId = bat(script: "docker ps -a -f publish=${port} -q", returnStdout: true).trim().readLines().drop(1).join('')
                            if (env.containerId != '') {
                                echo "Stopping and removing container running on ${port}"
                                bat "docker stop $env.containerId"
                                bat "docker rm $env.containerId"
                            } else {
                                echo "No container running on ${port} port."
                            }
                        }
                    }
                }

                stage ("Push to Docker") {
                    when{
                        expression {false}
                    }
                    steps {
                        echo "Push to Docker step"
                         bat "docker tag i-${userName}-${BRANCH_NAME}:${BUILD_NUMBER} ${registry}:i-${userName}-${BRANCH_NAME}:${BUILD_NUMBER}"
                         bat "docker tag i-${userName}-${BRANCH_NAME}:${BUILD_NUMBER} ${registry}:i-${userName}-${BRANCH_NAME}:latest"

                        bat "docker push ${registry}:i-${userName}-${BRANCH_NAME}:${BUILD_NUMBER}"
                        bat "docker push ${registry}:i-${userName}-${BRANCH_NAME}:latest"
                    }
                }
            }
        }        

        // stage('Kubernetes Deployment') {
		 // steps{
		     // bat "kubectl apply -f deployment.yaml"
		 // }
		//}
   	 }

	 post { 
			always { 
				echo 'Workspace Cleanup'
				cleanWs()
			}
		}
}
