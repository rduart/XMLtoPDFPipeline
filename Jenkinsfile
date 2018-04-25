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
				git url: 'https://github.com/rduart/XMLtoPDF.git', branch: 'Dev'
				
//				git url: 'git@github.com:rduart/XMLtoPDF.git', credentialsId: 'JenkinsPipelineID', branch: 'Dev'
				
				sh "git clean -f && git reset --hard origin/Dev"
			}
		}
/*		stage('Starting Build') {
            steps {
				slackSend (color: '#FFFF00', message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
			}
		} */
        stage('Clean') {
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
                    sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml -Dintegration-tests.skip=true -Dbuild.number=${targetVersion} clean package -B"
					
//					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml  -DreleaseVersion=${version} -DdevelopmentVersion=${pom.version} -DpushChanges=false -DlocalCheckout=true -DpreparationGoals=initialize release:prepare release:perform -B"

                    def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
                    // get the current development version 
                    developmentArtifactVersion = "${pom.version}-${targetVersion}"
					echo 'developmentArtifactVersion:   ' + developmentArtifactVersion
                    print pom.version
                }

            }
        }
		stage('Branch') {
			steps {
				script {
					def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
					echo pom.toString
					def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")
					def user = 'rduart@trinitytg.com'
					def pass = 'Trinity$007'
					
					print pom.version
					print version
					
//					echo "Here are the numbers; ${major}.${minor}.${build.number}"
//					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml  -Dusername=${user} -Dpassword=${pass} -DbranchName=${version} -DreleaseVersion=${version} -DdevelopmentVersion=${pom.version} -DautoVersionSubmodules=true -DupdateWorkingCopyVersions=true release:branch -B"
//					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml  -DbranchName=${version} -DreleaseVersion=${version} -DdevelopmentVersion=${pom.version} -DautoVersionSubmodules=true -DupdateWorkingCopyVersions=true release:branch -B"
		
				}
			}
		}
		
		stage('Prepare') {
			steps {
				script {
				
					def pom = readMavenPom file: '/var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml'
					def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")
					git url: 'https://github.com/rduart/XMLtoPDF.git', branch: ${version}
					
					
					sh "git clean -f && git reset --hard origin/${version}"
					sh "git checkout ${version}"
					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml -Dtag=${version} -DreleaseVersion=${version} -DdevelopmentVersion=${developmentVersion} release:prepare -B"
//					sh "'${mvnHome}/bin/mvn' -e --file /var/lib/jenkins/workspace/jiraPL/XMLtoPDF/pom.xml -DreleaseVersion=${releaseVersion} -DdevelopmentVersion=${developmentVersion} release:clean release:prepare release:perform -B"
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
    def pom = readMavenPom file: 'pom.xml'
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
