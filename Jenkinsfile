pipeline {
    // run on jenkins nodes that has no label
    agent any
    // global env variables
    environment {
        EMAIL_RECIPIENTS = 'mahmoud.romeh@test.com'
		mvnHome = tool 'Maven_Config' 
    }
    stages {

		stage('Preparation') {
			steps {
				git url: 'https://github.com/rduart/XMLtoPDF.git', branch: 'master'
				
				//This git is for SSH the JenkinsPipelineID is the name for Git
//				git url: 'git@github.com:rduart/XMLtoPDF.git', credentialsId: 'JenkinsPipelineID', branch: 'master'
				
				sh "git clean -f && git reset --hard origin/Dev"
			}
		}
/*		stage('Starting Build') {
            steps {
				slackSend (color: '#FFFF00', message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
			}
		} */
        stage('Clean Build') {
            steps {
                // Run the maven build
                script {
                    // Get the Maven tool.
                    // ** NOTE: This 'M3' Maven tool must be configured
                    // **       in the global configuration.
                    echo 'Pulling...' + env.BRANCH_NAME
					
                    def targetVersion = getDevVersion()
                    print 'target build version...'
                    print targetVersion
					
                    sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml -Dintegration-tests.skip=true -Dbuild.number=${targetVersion} clean package cobertura:cobertura -Dcobertura.report.format=xml -B"
				
                    def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
                    // get the current development version 
                    developmentArtifactVersion = "${pom.version}-${targetVersion}"
					echo 'developmentArtifactVersion:   ' + developmentArtifactVersion
                    print pom.version
                }

            }
        }
// build new branch is the code that commented out.		
/*		stage('Build Release') {
			steps {
				script {
				
					def targetVersion = getReleaseVersion()
					def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
					def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")
					
					print targetVersion
					print pom.version
					print version
					
					echo "Here are the numbers; ${major}.${minor}.${build.number}"
					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml  -DbranchName=${version} -DreleaseVersion=${version} -DdevelopmentVersion=${pom.version} -DautoVersionSubmodules=true -DupdateWorkingCopyVersions=true release:branch -B"
		
				}
			}
		} */
		
		stage('Build Release') {
			steps {
				script {
					def targetVersion = getReleaseVersion()
					def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
					def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")
					
					print targetVersion
					print pom.version
					print version
					
					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml -Dtag=${version} -DreleaseVersion=${version} -DdevelopmentVersion=${version} release:prepare release:perform -B"

				}
			}
		}
		stage('SonarQube analysis') { 
    		steps { 
				withSonarQubeEnv('SonarQubeServer') {
					sh '/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/Sonar/bin/sonar-scanner' +
					' -Dsonar.host.url=http://158.96.16.211:9000/' + 
					' -Dsonar.projectVersion=1.0' +
					' -Dsonar.sourceEncoding=UTF-8' +
					' -Dsonar.projectKey=TestPipeline' +
					' -Dsonar.java.binaries=/var/lib/jenkins/workspace/TestPipeline/SpringPOC/target/classes' +
					' -Dsonar.sources=SpringPOC/src' +
					' -Dsonar.projectBaseDir=/var/lib/jenkins/workspace/TestPipeline'
				}
			}
			post {
                always {
                    echo 'SonarQube Analysis  Done'
                }
				failure {
					echo 'SonarQube Analysis  failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - SonarQube Analysis Failed -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - SonarQube Analysis Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'SonarQube Analysis Success'
				}	
			}
		} 
    	stage('SonarQube Quality Gate') { 
			steps {
				node('master'){ 
					script {
						timeout(time: 1, unit: 'HOURS') { 
							echo '************ Inside Quality Gate'
							qualityGate = waitForQualityGate() 
							echo qualityGate.status.toString() 
							if (qualityGate.status != 'OK') {
							
								testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Sonar Quality Gate Failure -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

								response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

								echo response.successful.toString()
								echo response.data.toString()
						
								slackSend (color: '#FFFF00', message: "Failed: Job - Sonar Quality Gate Failure '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
								error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
							}
						}
					}
				}
			}
			post {
                always {
                    echo 'SonarQube Quality Gate  Done'
                }
				failure {
					echo 'SonarQube Quality Gate  failure'
				}
				success {
					echo 'SonarQube Quality Gate Success'
				}	
			}
		}
		stage('Unit Test Report') {   
            steps {
				junit '**/target/surefire-reports/*.xml'
	    	}     
        	post {
				always {
                 	echo 'always'
                }
				changed {
		 			echo 'change'
				}
				aborted {
					echo 'aborted'
				}
				failure {
					echo 'failure'
				}
				success {
					echo 'success'
				}
				unstable {
					echo 'unstable'
				}
            }
        } 
		stage('Code Coverage Report') {   
            steps {
				cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/target/site/cobertura/coverage.xml', conditionalCoverageTargets: '70, 0, 0', failUnhealthy: false, failUnstable: false, lineCoverageTargets: '80, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '80, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
	    	}
			post {
                always {
                    echo 'Code Coverage Report  Done'
                }
				failure {
					echo 'Code Coverage Report  failure'
				}
				success {
					echo 'Code Coverage Report Success'
				}	
			}
		} 
		stage('Maven Nexus Deploy') {
			steps {
				sh "'${mvnHome}/bin/mvn' -X -B --file /var/lib/jenkins/workspace/TestPipeline/SpringPOC/pom.xml -Dintegration-tests.skip=true deploy"
            }
			post {
                always {
                   echo 'Maven Nexus Deploy  Done'
                }
				failure {
					echo 'Maven Nexus Deploy  failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Nexus Depolyment Failed -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - AWS Code Deploy Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'Maven Nexus Deploy Success'
				}	
			}
		} 
		stage('Jira Update Issues') {
			steps {
				echo 'Jira Update Issues'
				
				step([$class: 'hudson.plugins.jira.JiraIssueUpdater', 
					issueSelector: [$class: 'hudson.plugins.jira.selector.DefaultIssueSelector'], 
					scm: [$class: 'GitSCM', branches: [[name: '*/master']], 
					userRemoteConfigs: [[url: 'https://github.com/CA-MMISDigitalServices/Dev.git']]]])
			
			}
			post {
                always {
					echo 'Jira Update Issues'
                }	
				failure {
					echo 'Jira Update Issues  failure'
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Jira Update Issues Failed-  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Jira Update Issues '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'Jira Update Issues Success'
				}
			}		
		} 
		stage('Security Dependency Check') {
			steps {
				echo 'Security Dependency Check'
				dependencyCheckAnalyzer datadir: '', hintsFile: '', includeCsvReports: false, includeHtmlReports: false, includeJsonReports: false, includeVulnReports: false, isAutoupdateDisabled: false, outdir: '', scanpath: '', skipOnScmChange: false, skipOnUpstreamChange: false, suppressionFile: '', zipExtensions: ''
			}
			post {
                always {
					echo 'Security Dependency Check'
                }	
				failure {
					script {
						echo 'Security Dependency Check  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Security Dependency Check -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Security Dependency Check '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'Security Dependency Check Success'
				}
			}
		}
 		stage('Security Dependency Publisher') {
			steps {
				echo 'Security Dependency Check'
				dependencyCheckPublisher canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '', unHealthy: ''
			}
			post {
                always {
					echo 'Security Dependency Publisher'
                }	
				failure {
					script {
						echo 'Security Dependency Publisher  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - Security Dependency Check -  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - Security Dependency Publisher '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'Security Dependency Publisher Success'
				}
			}
		} 
		stage('AWS Code Deploy') {
			steps {
				echo 'AWS Code Deploy'
				echo "env.AWS_ACCESS_KEY_ID :" + env.AWS_ACCESS_KEY_ID
				echo "env.AWS_SECRET_ACCESS_KEY :" + env.AWS_SECRET_ACCESS_KEY
				
				step([$class: 'AWSCodeDeployPublisher', 
						applicationName: 'SpringPOC', 
						awsAccessKey: env.AWS_ACCESS_KEY_ID,
						awsSecretKey: env.AWS_SECRET_ACCESS_KEY, 
						credentials: 'awsAccessKey', 
						deploymentConfig: 'CodeDeployDefault.OneAtATime', 
						deploymentGroupAppspec: false, 
						deploymentGroupName: 'SpringPOCDG', 
						deploymentMethod: 'deploy', 
						excludes: '', 
						iamRoleArn: '', 
						includes: '**', 
						pollingFreqSec: 15, 
						pollingTimeoutSec: 300, 
						proxyHost: '', 
						proxyPort: 0, 
						region: 'us-gov-west-1', 
						s3bucket: 'codedeploybucket', 
						s3prefix: '', 
						subdirectory: 'SpringPOC', 
						versionFileName: '', 
						waitForCompletion: true])
			
			}
			post {
                always {
					echo 'AWS Code Deploy'
                }	
				failure {
					script {
						echo 'AWS Code Deploy  failure'
						testIssue = [fields: [ project: [key: 'PTP'],
									summary: 'Jenkins Build Failure.',
									description: "Jenkins Build Failed - AWS Code Deploy Failed-  Job name: '${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}  URL: ${env.BUILD_URL}'",
									priority: [name: 'Highest'],
									issuetype: [name: 'Bug']]]

						response = jiraNewIssue issue: testIssue, site: 'CAMMIS'

						echo response.successful.toString()
						echo response.data.toString()
						
						slackSend (color: '#FFFF00', message: "Failed: Job - AWS Code Deploy Failed '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
								// Fail the build
//						error "Pipeline aborted due to quality gate failure "
					}
				}
				success {
					echo 'AWS Code Deploy Success'
					slackSend (color: '#00FF00', message: "Code Deploy SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
				}
			}		
		}
	}
}


def developmentArtifactVersion = ''

def getDevVersion() {
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    print 'build  versions...'
    print versionNumber
    return versionNumber
}


def getReleaseVersion() {
    def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    return pom.version.replace("-SNAPSHOT", ".${versionNumber}")
}

def sendEmail(status) {
    mail(
            to: "$EMAIL_RECIPIENTS",
            subject: "Build $BUILD_NUMBER - " + status + " (${currentBuild.fullDisplayName})",
            body: "Changes:\n " + getChangeString() + "\n\n Check console output at: $BUILD_URL/console" + "\n")
}

def releasedVersion = ''
// get change log to be send over the mail
//@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += " - ${truncated_msg} [${entry.author}]\n"
        }
    }

    if (!changeString) {
        changeString = " - No new changes"
    }
    return changeString
}
