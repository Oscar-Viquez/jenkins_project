import groovy.json.JsonBuilder

pipeline {
	agent any

	// Environment Variables
	environment {
		// Notifications
		EMAIL_SUCCESS = 'oviquez@auxis.com'
		EMAIL_FAILURE = 'oviquez@auxis.com'
		EMAIL_REVIEW_APPROVAL = 'oviquez@auxis.com'
		EMAIL_DEPLOY_STAGE_COMPLETED = 'oviquez@auxis.com'
		EMAIL_DEPLOY_PRODUCTION_APPROVAL = 'oviquez@auxis.com'
		EMAIL_DEPLOY_PRODUCTION_COMPLETED = 'oviquez@auxis.com'
		
		// Flags to adjust the behavior of the pipeline
		EXECUTE_STAGE_REVIEW = 'false'											// Flags if the process should be reviewed using the Workflow Analyzer
		EXECUTE_STAGE_TEST = 'false'												// Run the test phase
		DEPLOY_TO_PRODUCTION = 'true'											// Determines if the package should be deployed in Production
		CREATE_ASSETS_AND_QUEUES = 'false'										// Indicates if the process to create assets and queues should be executed
		
		WORKFLOW_ANALYZER_THRESHOLD = '98'										// Minimum score required to approve the project automatically [0-100]
		TEST_TESTSET = 'TestSet'												// Name of the TestSet to execute for the process
		ASSETS_AND_QUEUES_PROCESS = 'VMware-AssetsAndQueuesDefinition'			// Process to execute to create Assets and Queues
		ASSETS_AND_QUEUES_FILE = "$WORKSPACE\\Input\\AssetQueueDefinition.xlsx"	// Location of the assets and queue definition file
		
		// Stage Orchestrator
		UIPATH_ORCH_STAGE_URL = 'https://cloud.uipath.com/'
		UIPATH_ORCH_STAGE_LOGICAL_NAME = 'auxis'
		UIPATH_ORCH_STAGE_TENANT = 'Default'
		UIPATH_ORCH_STAGE_FOLDER = 'Default'
		UIPATH_ORCH_STAGE_CREDENTIALS = 'UiPathCloudAPIKey'
		
		// Production Orchestrator
		UIPATH_ORCH_PRODUCTION_URL = 'https://devrpa01.auxis-its.com/'
		UIPATH_ORCH_PRODUCTION_TENANT = 'AuxisTraining'
		UIPATH_ORCH_PRODUCTION_FOLDER = 'Production'
		UIPATH_ORCH_PRODUCTION_CREDENTIALS = 'UiPathDevAuxisTraining'

		// Version numbers
		MAJOR = '1'
		MINOR = '0'
		
		// Review phase
		WORKFLOW_ANALYZER_PROCESS = 'Automated_Workflow_Analyzer'				// Name of the RPA process that runs the Workflow Analyzer
		WORKFLOW_ANALYZER_RESULTS = "Workflow Analyzer Output.xlsx"				// File containing the results of the Workflow Analyzer process

		WORKFLOW_ANALYZER_PARAMETERS = "$WORKSPACE\\WAParameters.json"			// INTERNAL USE: file with the parameters required for the Workflow Analyzer
		WORKFLOW_ANALYZER_SCORE = '0'											// INTERNAL USE: Score obtained after running the Workflow Analyzer process
		
		// Test phase
		TEST_RESULTS = 'TestResults.xml'										// Name of the file that stores the results of the test
		
		// Deploy phase (Stage and Prod)
		ASSETS_AND_QUEUES_PARAMETERS = "$WORKSPACE\\AssetsParams.json"
		
		// Paths
		UIPATH_PACKAGE = "Output\\${env.BUILD_NUMBER}"							// Determines where to store the new version of the code
		PROJECT_ENTRY_POINT = "Main.xaml"										// Entry point of the automation, it is "Main.xaml" by default (case sensitive)
	}

	// Stages of the pipeline
	stages {
		// Get ready for the deployment, get the latest version of the project
		stage('Preparing') {
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				
				// Clean before build
                cleanWs()
				
				echo "$STAGE_NAME - Jenkins Home: ${env.JENKINS_HOME}"
				echo "$STAGE_NAME - Jenkins URL: ${env.JENKINS_URL}"
				echo "$STAGE_NAME - Jenkins JOB Number: ${env.BUILD_NUMBER}"
				echo "$STAGE_NAME - Jenkins JOB Name: ${env.JOB_NAME}"
				echo "$STAGE_NAME - Jenkins Workspace: ${env.WORKSPACE}"
				echo "$STAGE_NAME - GitHub BranchName: ${env.BRANCH_NAME}"
				echo "$STAGE_NAME - UiPath package: ${UIPATH_PACKAGE}"
				checkout scm
			}
		}

		
		// Run the Workflow Analyzer process and evaluate the results
		stage('Review') {
			when {environment(name: "EXECUTE_STAGE_REVIEW", value: "true")}
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				
				// Create the parameters file, to send the path of the project
				echo "$STAGE_NAME - Creating parameters file for the Workflow Analyzer process"
				script {
					def json = new groovy.json.JsonBuilder()
					def paramsMap = [
						in_DesignerEmail: "",
						in_DesignLeadEmail: "",
						in_CodeRepositoryAddress: "$WORKSPACE"
					]
					json "parameters": paramsMap
					def file = new File("${WORKFLOW_ANALYZER_PARAMETERS}")
					file.write(groovy.json.JsonOutput.prettyPrint(json.toString()))
				}
				
				// Run the Workflow Analyzer process
				echo "$STAGE_NAME - Executing the Workflow Analyzer process"
				UiPathRunJob(
					orchestratorAddress: "${UIPATH_ORCH_STAGE_URL}",
					orchestratorTenant: "${UIPATH_ORCH_STAGE_TENANT}",
					credentials: Token(accountName: "${UIPATH_ORCH_STAGE_LOGICAL_NAME}", credentialsId: "${UIPATH_ORCH_STAGE_CREDENTIALS}"),
					folderName: "${UIPATH_ORCH_STAGE_FOLDER}",
					processName: "${WORKFLOW_ANALYZER_PROCESS}",
					parametersFilePath: "${WORKFLOW_ANALYZER_PARAMETERS}",
					jobType: [$class: 'UnattendedJobTypeEntry'],
					waitForJobCompletion: true,
					failWhenJobFails: true,
					timeout: 3600
				)

				// Evaluate the results
				echo "$STAGE_NAME - Evaluating the results of the Worklow Analyzer process"
				script {
					def data = readFile(file: 'ProjectQualityScore.txt')
					WORKFLOW_ANALYZER_SCORE = data.toString().replaceAll('%', '')
					println("... Workflow analyzer score: " + WORKFLOW_ANALYZER_SCORE + ', threshold: ' + WORKFLOW_ANALYZER_THRESHOLD)
					
					// Validate if the score met the threshold
					if (Double.parseDouble(WORKFLOW_ANALYZER_SCORE) < Double.parseDouble(WORKFLOW_ANALYZER_THRESHOLD))
					{
						println('... The process did not pass the Workflow Analyzer threshold')
						WORKFLOW_ANALYZER_APPROVED = "false"
						
						// Email for the developer
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Failed (Review)",
							to: "${EMAIL_SUCCESS}",
							body: "<p>A review completed for the project using the Workflow Analyzer but the current score (${WORKFLOW_ANALYZER_SCORE}%) is below the threshold (${WORKFLOW_ANALYZER_THRESHOLD}%). Attached you can find a copy of the report for your reference.</p><p>The project was put on-hold and requires a manual approval to be able to continue with the deployment; an email will be sent to the approver for review.</p>",
							attachmentsPattern: "${WORKFLOW_ANALYZER_RESULTS}";

						// Email for the approver
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Approval Required (Review)",
							to: "${EMAIL_REVIEW_APPROVAL}",
							body: "<p>A review completed for the project using the Workflow Analyzer but the current score (${WORKFLOW_ANALYZER_SCORE}%) is below the threshold (${WORKFLOW_ANALYZER_THRESHOLD}%). Attached you can find a copy of the report for your reference.</p><p>The project was put on-hold and requires a manual approval to be able to continue with the deployment. Click this <a href='${BUILD_URL}'>link</a> to view the project details and either approve or abort the deployment.</p>",
							attachmentsPattern: "${WORKFLOW_ANALYZER_RESULTS}";

						// Display an input in Jenkins to ask for approval
						echo "$STAGE_NAME - Waiting for approval before moving forward"
						input (
							message: "The Workflow Analyzer score (${WORKFLOW_ANALYZER_SCORE}%) is below the threshold (${WORKFLOW_ANALYZER_THRESHOLD}%). Do you want to continue with the pipeline?"
						)
						
						// Email confirming the process was approved (if it got to this point is because it was not aborted)
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Completed (Review)",
							to: "${EMAIL_SUCCESS}",
							body: "<p>The results of the Workflow Analyzer (${WORKFLOW_ANALYZER_SCORE}%) were reviewed and approved by the administrator.</p><p>The process can now continue with the next stage of the CI/CD pipeline.</p>";
					}
					else
					{
						// The Workflow Analyzer succeded, the process can continue without the need for a manual approval
						println('... The process passed the Workflow Analyzer threshold')
						WORKFLOW_ANALYZER_APPROVED = "true"
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Completed (Review)",
							to: "${EMAIL_SUCCESS}",
							body: "<p>A review completed for the project using the Workflow Analyzer and the score (${WORKFLOW_ANALYZER_SCORE}%) met the threshold (${WORKFLOW_ANALYZER_THRESHOLD}%). Attached you can find a copy of the report for your reference.</p><p>No action is required and the project will continue with the next stage of the CI/CD pipeline.</p>",
							attachmentsPattern: "${WORKFLOW_ANALYZER_RESULTS}";
					}
				}
			}
		}
		
		
		// Create the NUPKG for the process
		stage('Pack') {
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				echo "$STAGE_NAME - Packing the project"
				UiPathPack (
					outputPath: "${UIPATH_PACKAGE}",
					projectJsonPath: "project.json",
					version: [$class: 'ManualVersionEntry', version: "${MAJOR}.${MINOR}.${env.BUILD_NUMBER}"],
					useOrchestrator: false,
					traceLevel: "None"
				)
			}
		}
		
		
		// Deploy the project to Stage
		stage('Deploy to Stage') {
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				echo "$STAGE_NAME - UiPath Stage Orchestrator URL ${UIPATH_ORCH_STAGE_URL}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Tenant ${UIPATH_ORCH_STAGE_TENANT}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Folder ${UIPATH_ORCH_STAGE_FOLDER}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Credentials ${UIPATH_ORCH_STAGE_CREDENTIALS}"
				
				script {
					if ("${CREATE_ASSETS_AND_QUEUES}" == 'true') {
						println("$STAGE_NAME - Created assets and queues")
						println("$STAGE_NAME - Using assets and queues file: $ASSETS_AND_QUEUES_FILE")
						
						// Determine the type of Orchestrator
						def orchestratorType = ''
						
						if ("$UIPATH_ORCH_STAGE_URL".contains("cloud.uipath.com"))
						{
							orchestratorType = 'CLOUD'
						} else {
							orchestratorType = 'ON PREMISE'
						}
						
						// Create file with parameters
						//def json = new groovy.json.JsonBuilder()
						//def paramsMap = [
						//	in_OrchestratorQueueName: "",
						//	in_MaxNumberOfTransactions: 500,
						//	in_Definition_FilePath: "$ASSETS_AND_QUEUES_FILE",
						//	in_orchestrator_Type: "$orchestratorType",
						//	in_orchestrator_Environment: "STAGE"
						//]
						//json "InputArguments": paramsMap
						//def file = new File("$ASSETS_AND_QUEUES_PARAMETERS")
						//file.write(groovy.json.JsonOutput.prettyPrint(json.toString()))
						
						// Run the assets and queues process
						//UiPathRunJob(
						//	orchestratorAddress: "${UIPATH_ORCH_STAGE_URL}",
						//	orchestratorTenant: "${UIPATH_ORCH_STAGE_TENANT}",
						//	credentials: Token(accountName: "${UIPATH_ORCH_STAGE_LOGICAL_NAME}", credentialsId: "${UIPATH_ORCH_STAGE_CREDENTIALS}"),
						//	folderName: "${UIPATH_ORCH_STAGE_FOLDER}",
						//	processName: "${ASSETS_AND_QUEUES_PROCESS}",
						//	parametersFilePath: "${ASSETS_AND_QUEUES_PARAMETERS}",
						//	jobType: [$class: 'UnattendedJobTypeEntry'],
						//	waitForJobCompletion: true,
						//	failWhenJobFails: true,
						//	timeout: 3600
						//)
					}
				}
				
				// Deploy the process to Stage
				echo "$STAGE_NAME - Deploying package to Stage"
				UiPathDeploy (
					orchestratorAddress: "${UIPATH_ORCH_STAGE_URL}",
					orchestratorTenant: "${UIPATH_ORCH_STAGE_TENANT}",
					credentials: Token(accountName: "${UIPATH_ORCH_STAGE_LOGICAL_NAME}", credentialsId: "${UIPATH_ORCH_STAGE_CREDENTIALS}"),
					folderName: "${UIPATH_ORCH_STAGE_FOLDER}",
					packagePath: "${UIPATH_PACKAGE}",
					traceLevel: 'None',
					environments: '',
					entryPointPaths: "${PROJECT_ENTRY_POINT}"
				)
				
				// Notify the user when deployment to Stage is complete
				echo "$STAGE_NAME - Sending email after deploying to Stage"
				emailext mimeType: 'text/html',
					subject: "${currentBuild.fullDisplayName} - Completed (${LAST_STAGE})",
					to: "${EMAIL_DEPLOY_STAGE_COMPLETED}",
					body: "<p>The RPA process has been successfully deployed to the Stage environment.</p><ul><li><b>Orchestrator URL:</b> ${UIPATH_ORCH_STAGE_URL}${UIPATH_ORCH_STAGE_LOGICAL_NAME}</li><li><b>Tenant:</b> ${UIPATH_ORCH_STAGE_TENANT}</li><li><b>Folder:</b> ${UIPATH_ORCH_STAGE_FOLDER}</li></ul>";
			}
		}
		
		
		// Run the Test Set
		stage('Run Tests') {
			when {environment(name: "EXECUTE_STAGE_TEST", value: "true")}
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				
				// Run the Test Set
				echo "$STAGE_NAME -Executing Test Set ${TEST_TESTSET}"				
				UiPathTest (
					testTarget: [$class: 'TestSetEntry', testSet: "${TEST_TESTSET}"],
					orchestratorAddress: "${UIPATH_ORCH_STAGE_URL}",
					orchestratorTenant: "${UIPATH_ORCH_STAGE_TENANT}",
					folderName: "${UIPATH_ORCH_STAGE_FOLDER}",
					timeout: 10000,
					testResultsOutputPath: "${TEST_RESULTS}",
					credentials: Token(accountName: "${UIPATH_ORCH_STAGE_LOGICAL_NAME}", credentialsId: "${UIPATH_ORCH_STAGE_CREDENTIALS}"),
					parametersFilePath: '',
					traceLevel: 'None'
				)
				
				// Notify the user when testing is complete
				script {
					println(".. Evaluating test results: $currentBuild.currentResult")
					if (currentBuild.currentResult == 'UNSTABLE') {
						// At least one Test Case failed, marking the deploy as FAILURE
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Failed ($LAST_STAGE)",
							to: "${EMAIL_FAILURE}",
							body: '<p>At least one Test Case was marked as failed while testing the process.</p><p>Attached you can find the results of the testing.</p>',
							attachmentsPattern: "${TEST_RESULTS}";
						currentBuild.result = 'FAILURE'
						exit 1
					} else {
						// Testig completed, process can continue automatically
						emailext mimeType: 'text/html',
							subject: "${currentBuild.fullDisplayName} - Completed (${LAST_STAGE})",
							to: "${EMAIL_SUCCESS}",
							body: "<p>Testing for the process has completed using the <b>$TEST_TESTSET</b> Test Set.</p><p>Attached you can find the results of the testing.</p>",
							attachmentsPattern: "${TEST_RESULTS}";
					}
				}
			}
		}
		
		
		// Ask for approval before moving forward with the deployment
		stage ('Approval for Production') {
			when {environment(name: "DEPLOY_TO_PRODUCTION", value: "true")}
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}

				script {
					// Determine which files to include
					def attachments = ''
					
					println("$EXECUTE_STAGE_REVIEW")
					println("$EXECUTE_STAGE_TEST")
					
					if("$EXECUTE_STAGE_REVIEW" == 'true') {
						attachments += ",$WORKFLOW_ANALYZER_RESULTS"
					}
					
					if ("$EXECUTE_STAGE_TEST" == 'true') {
						attachments += ",$TEST_RESULTS"
					}
					
					// Notify the approver when the deployment is ready to be moved to Production
					println("$STAGE_NAME - Sending email asking for approval")
					emailext mimeType: 'text/html',
						subject: "${currentBuild.fullDisplayName} - Ready for Production",
						to: "${EMAIL_DEPLOY_PRODUCTION_APPROVAL}",
						body: "<p>The RPA process is ready to be promoted to Production, it has completed all the previous steps.</p><p>The project was put on-hold and requires a manual approval to be able to continue with the deployment. Click this <a href='${BUILD_URL}'>link</a> to view the project details and either approve or abort the deployment.</p>",
						attachmentsPattern: "$attachments";
					
				}
				

				// Display an input in Jenkins to ask for approval
				echo "$STAGE_NAME - Waiting for approval before moving forward"
				input (
					message: "The RPA process is ready to be promoted to Production. Do you want to continue with the pipeline?",		
					submitter: "viquezo";

				)

				// Notify the developer that the process was approved for Production (if it gets to this part is because it was approved)
				echo "$STAGE_NAME - Sending email asking for approval"
				emailext mimeType: 'text/html',
					subject: "${currentBuild.fullDisplayName} - Success (${STAGE_NAME})",
					to: "${EMAIL_SUCCESS}",
					body: "<p>The deployment of the RPA process to Production has been approved, the process is going to continue with the next stage of the CI/CD pipeline.</p>";
			}
		}
		
		
		// Deploy the project to Production
		stage('Deploy to Production') {
			when {environment(name: "DEPLOY_TO_PRODUCTION", value: "true")}
			steps {
				script {
					LAST_STAGE=env.STAGE_NAME
					println("Starting the ${STAGE_NAME} stage")
				}
				echo "$STAGE_NAME - UiPath Stage Orchestrator URL ${UIPATH_ORCH_PRODUCTION_URL}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Tenant ${UIPATH_ORCH_PRODUCTION_TENANT}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Folder ${UIPATH_ORCH_PRODUCTION_FOLDER}"
				echo "$STAGE_NAME - UiPath Stage Orchestrator Credentials ${UIPATH_ORCH_PRODUCTION_CREDENTIALS}"
				
				// Deploy the process to Production
				echo "$STAGE_NAME - Deploying package to Production"
				UiPathDeploy (
					orchestratorAddress: "${UIPATH_ORCH_PRODUCTION_URL}",
					orchestratorTenant: "${UIPATH_ORCH_PRODUCTION_TENANT}",
					folderName: "${UIPATH_ORCH_PRODUCTION_FOLDER}",
					credentials: [$class: 'UserPassAuthenticationEntry', credentialsId: "${UIPATH_ORCH_PRODUCTION_CREDENTIALS}"],
					packagePath: "${UIPATH_PACKAGE}",
					traceLevel: 'None',
					environments: '',
					entryPointPaths: "${PROJECT_ENTRY_POINT}"
				)
				
				// Notify the user when deployment to Production is complete
				echo "$STAGE_NAME - Sending email after deploying to Prodution"
				emailext mimeType: 'text/html',
					subject: "${currentBuild.fullDisplayName} - Completed (${LAST_STAGE})",
					to: "${EMAIL_DEPLOY_PRODUCTION_COMPLETED}",
					body: "<p>The RPA process has been successfully deployed to the Production environment.</p><ul><li><b>Orchestrator URL:</b> ${UIPATH_ORCH_PRODUCTION_URL}</li><li><b>Tenant:</b> ${UIPATH_ORCH_PRODUCTION_TENANT}</li><li><b>Folder:</b> ${UIPATH_ORCH_PRODUCTION_FOLDER}</li></ul>";
			}
		}
	}
	
	// Post build activities
	post {
		failure {
			echo "Pipeline failed to stage ${LAST_STAGE}"
			emailext mimeType: 'text/html',
				subject: "${currentBuild.fullDisplayName} - Failed (${LAST_STAGE})",
				to: "${EMAIL_FAILURE}",
				body: "<p>An  error occurred while running the CI/CD pipeline in the <b>${LAST_STAGE}</b> stage.</p><p>The process has been stopped.</p>";
		}
		unstable {
			echo "Pipeline was marked as unstable in stage ${LAST_STAGE}"
			emailext mimeType: 'text/html',
				subject: "${currentBuild.fullDisplayName} - Failed ($LAST_STAGE)",
				to: "${EMAIL_FAILURE}",
				body: "<p>The deployment was marked as <b>Unstable</b> in the <b>${LAST_STAGE}</b> stage.</p><p>The process has been stopped.</p>";
		}
		aborted {
			echo "Aborted execution in stage ${LAST_STAGE}"
			emailext mimeType: 'text/html',
				subject: "${currentBuild.fullDisplayName} - Aborted (${LAST_STAGE})",
				to: "${EMAIL_FAILURE}",
				body: "<p>The pipeline has been aborted by the administrator while running the <b>${LAST_STAGE}</b> stage.</p>";
		}
		success {
			echo "Deployment completed successfully"
			emailext mimeType: 'text/html',
				subject: "${currentBuild.fullDisplayName} - Completed",
				to: "${EMAIL_SUCCESS}",
				body: '<p>All the stages in the CI/CD pipeline completed successfully.</p>';
		}
	}
}

// Pending
// - Create assets in Stage
// - Send email after creating assets in Stage
// - Create assets in production
// - Send email after creating assets in Production
// - Include the project name in the input message
