// Before you start...follow instructions here to setup script and docker plugin
// https://github.com/sonatype-nexus-community/nexus-ci-examples

//Create a Jenkins pipeline build with "Project is parameterised" and declare the following string settings.
  // "iqAppID"     - DESCRIPTION: IQ Server Application ID to evaluate against
  // "iqStage"     - DESCRIPTION: IQ Server stage to evaluate against, Options are: "build | stage-release | release"
  // "deploy_repo" - DESCRIPTION: Deployment repository for your built artefact. Usually "maven-releases"
  // "groupId"     - DESCRIPTION: groupId taken from the project pom.xml
  // "artifactId"  - DESCRIPTION: artifactId taken from the project pom.xml
  // "version"     - DESCRIPTION: version taken from the project pom.xml
  // "packaging"   - DESCRIPTION: The file format extension of the final artefact EG "ear | war | jar"

pipeline {
    agent {
       label 'gradle-node'
        }
    environment {
       ARTEFACT_NAME = "${WORKSPACE}/target/${artifactId}-${version}.${packaging}"
       TAG_FILE = "${WORKSPACE}/tag.json"
       IQ_SCAN_URL = ""
       iqStage = "${iqStage}"
    }

    stages {
        stage('Build') {
            steps {			
				// Performing build
				echo "Performing build"
				sh 'yes | sdkmanager --sdk_root=${ANDROID_HOME} --licenses && export ANDROID_SDK_ROOT=/home/jenkins/ && bash ./gradlew --no-daemon cyclonedxBom'
           }
        }

        // Once you run this pipeline once, you will need to approve the script from the console output
        stage('Nexus IQ Scan'){
            steps {
                script{         
                    try {
                        def policyEvaluation = nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: selectedApplication("${iqAppID}"), iqScanPatterns: [[scanPattern: "**/*.${packaging}"]], iqStage: "${iqStage}", jobCredentialsId: ''
                        echo "Nexus IQ scan succeeded: ${policyEvaluation.applicationCompositionReportUrl}"
                        IQ_SCAN_URL = "${policyEvaluation.applicationCompositionReportUrl}"
                    } 
                    catch (error) {
                        def policyEvaluation = error.policyEvaluation
                        echo "Nexus IQ scan vulnerabilities detected', ${policyEvaluation.applicationCompositionReportUrl}"
                        throw error
                    }
                }
            }
        }

        stage('Create tag'){
            steps {
                script {
    
                    // Git data (Git plugin)
                    echo "${GIT_URL}"
                    echo "${GIT_BRANCH}"
                    echo "${GIT_COMMIT}"
                    echo "${WORKSPACE}"

                    
                    // construct the meta data (Pipeline Utility Steps plugin)
                    def tagdata = readJSON text: '{}' 
                    tagdata.buildNumber = "${BUILD_NUMBER}" as String
                    tagdata.buildId = "${BUILD_ID}" as String
                    tagdata.buildJob = "${JOB_NAME}" as String
                    tagdata.buildTag = "${BUILD_TAG}" as String
                    tagdata.appVersion = "${version}" as String
                    tagdata.buildUrl = "${BUILD_URL}" as String
                    tagdata.iqScanUrl = "${IQ_SCAN_URL}" as String
                    tagdata.gitUrl = "${GIT_BRANCH}" as String
                    //tagData.promote = "no" as String

                    writeJSON(file: "${TAG_FILE}", json: tagdata, pretty: 4)
                    sh 'cat ${TAG_FILE}'

                    createTag nexusInstanceId: 'nexus', tagAttributesPath: "${TAG_FILE}", tagName: "${BUILD_TAG}"

                    // write the tag name to the build page (Rich Text Publisher plugin)
                    rtp abortedAsStable: false, failedAsStable: false, parserName: 'Confluence', stableText: "Nexus Repository Tag: ${BUILD_TAG}", unstableAsStable: true 
                }
            }
        }

        stage('Upload to Nexus Repository'){
            steps {
                script {
                    nexusPublisher nexusInstanceId: 'nexus', nexusRepositoryId: "${deploy_repo}", packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: "${packaging}", filePath: "${ARTEFACT_NAME}"]], mavenCoordinate: [artifactId: "${artifactId}", groupId: "${groupId}", packaging: "${packaging}", version: "${version}"]]], tagName: "${BUILD_TAG}"
                }
            }
        }
    }
}
