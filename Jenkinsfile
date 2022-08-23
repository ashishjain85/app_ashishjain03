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
        
    	stage ("Nuget restore") {
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
            when {
                branch "master"
            }
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

        stage('Kubernetes Deployment') {
            steps{
                //For master branch, publish before creating docker image
                script {
                    if (BRANCH_NAME == "master") {
                        bat "dotnet publish -c Release -o ${appName}/app/${userName}"
                    }
                }
                echo "Docker Image"
                bat "docker build -t i-${userName}-${BRANCH_NAME}:latest --no-cache -f Dockerfile ."

                echo "Push to Docker"
                bat "docker tag i-${userName}-${BRANCH_NAME}:latest ${registry}/i-${userName}-${BRANCH_NAME}:latest"
                bat "docker push ${registry}/i-${userName}-${BRANCH_NAME}:latest"

                echo "Execute kubectl"
                bat "kubectl apply -f deployment.yaml"
		    }
		}
   	 }

	 post { 
			always { 
				echo 'Workspace Cleanup'
				cleanWs()
			}
		}
}
