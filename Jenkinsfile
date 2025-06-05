// === Jenkinsfile (Scripted Pipeline) with English Comments ===

// === Global Variables ===
// def userConfig               // User configuration loaded from YAML
//def projectConfig            // Project configuration loaded from YAML
setNoti = ''                 // Notification webhook URL
stageName = ''              // Track current stage name
stageBuild = ''             // Build stage name from config
stageAnaly = ''             // Static analysis stage name from config
stageUtest = ''             // Unit test stage name from config
buildAgent = 'Server-PC'    // Default build agent (can be overridden by config)
sourceBuild = false         // Indicates if the build should proceed based on rules

// === Initial stage on specified agent ===
node(buildAgent) {
    echo "${buildAgent}"
    stageName = 'Init'
    stage(stageName) {
        initPipeline()  // Checkout SCM, read config files, validate build agent
    }
}

// === Main Pipeline Execution ===
node(buildAgent) {
    try {
        echo "${buildAgent}"

        stageName = 'Get Config'
        stage(stageName) {
            loadConfigs()           // Load user/project configuration
            validateBranchRules()   // Check if this branch is allowed to build
            prepareWorkspace()      // Prepare working directory and copy necessary files
        }

        // === Conditional Execution if build is allowed ===
        if (sourceBuild) {

            // === Build Stage ===
            if (stageBuild) {
                stageName = 'Build'
                stage(stageName) {
                    runBuild(cliConfig.build, stageBuild)  // Run build command
                }
            }

            // === Static Analysis Stage ===
            if (stageAnaly) {
                stageName = 'Static Analysis'
                stage(stageName) {
                    runStaticAnalysis(cliConfig.analysis, stageAnaly)  // Run static analysis tool
                }

                // === Cppcheck Report Generation ===
                if (stageAnaly == 'cppcheck') {
                    stageName = 'Generate Cppcheck Report'
                    stage(stageName) {
                        runCppcheckReport()  // Convert cppcheck XML to HTML and publish report
                    }
                }
            }

            // === Unit Testing Stage ===
            if (stageUtest) {
                stageName = 'Unit Test'
                stage(stageName) {
                    runUnitTest(cliConfig.test, stageUtest)  // Run unit tests
                }

                stageName = 'Coverage Result'
                stage(stageName) {
                    checkCoverage("coverage_${stageUtest}.yaml")  // Validate test coverage is 100%
                }

                stageName = 'Testlog Result'
                stage(stageName) {
                    checkTestLog("testlog_${stageUtest}.yaml")  // Check for failed tests
                }
            }
        }

        // === PR Merge Stage (only for pull request builds) ===
        if (env.CHANGE_ID) {
            stageName = 'Merge pull request'
            stage(stageName) {
                mergePullRequest()  // Call SCM API to merge pull request
            }
        }

        // === Final Result Archive Stage ===
        stage('Result URL') {
            generateResultZip()  // Compress result folder and archive the ZIP
        }

    } catch (err) {
        // === Error Handling ===
        currentBuild.result = 'FAILURE'
        echo "Error at stage : ${stageName}"
        throw err

    } finally {
        // === Notification Stage ===
        notifyResult()  // Send Microsoft Teams notification
    }
}

// === Function Definitions ===

def initPipeline() {
    cleanWs()                 // Clean workspace
    checkout scm              // Checkout source code from Git
    if (!env.GIT_URL) {
        env.GIT_URL = scm.userRemoteConfigs[0].url  // Set Git URL for later use
    }

    // Load YAML config files
    userConfig = readYaml file: "${env.WORKSPACE}\\jenkins\\user_config\\ci-config.yaml"
    projectConfig = readYaml file: "${env.WORKSPACE}\\jenkins\\project_config\\project_config.yaml"
    cliConfig = readYaml file: "${env.WORKSPACE}\\jenkins\\project_config\\runcommand.yaml"

    // Override build agent if configured
    buildAgent = userConfig.buildAgent ?: buildAgent
    echo "buildAgent : ${buildAgent}"
    echo "projectConfig.agentList : ${projectConfig.agentList}"

    // Check if the build agent is valid
    if (!projectConfig.agentList.contains(buildAgent)) {
        error "Invalid build agent: ${buildAgent}"
    }
}

def loadConfigs() {
    // Extract build/analysis/test targets from user config
    stageBuild = userConfig.buildSouce?.name
    echo "stageBuild : ${stageBuild}"
    stageAnaly = userConfig.analysis?.name
    echo "stageAnaly : ${stageAnaly}"
    stageUtest = userConfig.unitTest?.name
    echo "stageUtest : ${stageUtest}"

    // Set notification webhook URL
    def notiKey = userConfig.noti
    setNoti = projectConfig[notiKey] ?: error("Notification channel '${notiKey}' not found.")
    echo "setNoti : ${setNoti}"
}

def validateBranchRules() {
    // Allow only 'main' branch if configured
    def mainOnly = userConfig.onlymain
    if ((mainOnly == 'on' || mainOnly == true) && env.BRANCH_NAME != 'main') {
        error("Build only allowed on main branch.")
    }

    // Decide whether to build based on PR or commit
    def isPR = env.CHANGE_ID != null
    def isCommit = !isPR
    def buildRules = userConfig.buildRules ?: [userConfig.build]
    if ((isPR && buildRules.contains('pr')) || (isCommit && buildRules.contains('commit'))) {
        sourceBuild = true
    } else {
        error("Build rule does not allow this run.")
    }

    echo "sourceBuild = ${sourceBuild}"
}

def prepareWorkspace() {
    // Create result folder and copy runcommand files
    bat"""
        if not exist ${env.WORKSPACE}\\result mkdir ${env.WORKSPACE}\\result
        xcopy ${env.WORKSPACE}\\jenkins\\project_config\\runcommand\\* ${env.WORKSPACE}\\ /s /y
    """
}

def runBuild(targetList, targetName) {
    // Find and run build command
    def target = targetList.find { it.name == targetName }
    if (!target) error "Build target '${targetName}' not found"
    bat target.run
}

def runStaticAnalysis(targetList, targetName) {
    // Find and run static analysis command
    def target = targetList.find { it.name == targetName }
    if (!target) error "Static analysis target '${targetName}' not found"
    bat target.run
}

def runCppcheckReport() {
    // Generate HTML report from cppcheck results
    bat """
        "py" "C:\\Program Files\\Cppcheck\\cppcheck-htmlreport" --file=cppcheck-report.xml --report-dir=cppcheck-report --source-dir=.
    """
    recordIssues tools: [[ $class: 'CppCheck', pattern: 'cppcheck-report.xml' ]]
    publishHTML([reportDir: "cppcheck-report", reportFiles: 'index.html', reportName: 'Cppcheck Report'])
}

def runUnitTest(targetList, targetName) {
    // Find and run unit test command
    def target = targetList.find { it.name == targetName }
    if (!target) error "Unit test target '${targetName}' not found"
    bat target.run
}

def checkCoverage(filePath) {
    // Check if coverage is 100%
    if (!fileExists(filePath)) return
    def cov = readYaml file: filePath
    if (cov.coverage != 100.0) {
        error "Coverage is not 100%! Got ${cov.coverage}"
    }
}

def checkTestLog(filePath) {
    // Check if there are test failures
    if (!fileExists(filePath)) return
    def result = readYaml file: filePath
    if (result.tests) {
        error "Test failed: ${result.num}, ${result.tests}"
    }
}

def mergePullRequest() {
    // Perform PR merge using SCM-Manager API
    def prId = env.CHANGE_ID
    def gitUrl = env.GIT_URL ?: scm.userRemoteConfigs[0].url
    def firstPart = gitUrl.replaceAll('(/repo/.*)', '/')
    def secondPart = gitUrl.replaceAll('.*?repo/', '')
    def credId = projectConfig.credentialsId

    withCredentials([usernamePassword(credentialsId: credId, usernameVariable: 'SCM_USER', passwordVariable: 'SCM_PASS')]) {
        bat """
            curl -s -u %SCM_USER%:%SCM_PASS% ^
            -H "accept: */*" ^
            -H "Content-Type: application/vnd.scmm-mergeCommand+json;v=2" ^
            -d "{\\"commitMessage\\": \\"Merge by Jenkins\\", \\"shouldDeleteSourceBranch\\": false}" ^
            ${firstPart}api/v2/merge/${secondPart}/${prId}?strategy=MERGE_COMMIT
        """
    }
}

def generateResultZip() {
    // Compress result folder and archive ZIP file
    def timestamp = new Date().format('yyyyMMdd-HHmm')
    def zipName = "CI_${env.BRANCH_NAME}-${timestamp}.zip"
    bat "powershell Compress-Archive -Path result\\ ${zipName} -Update"
    echo "Result URL: ${env.JOB_URL}lastSuccessfulBuild/artifact/${zipName}"
    archiveArtifacts zipName
}

def notifyResult() {
    // Send Microsoft Teams notification with build result
    def status = currentBuild.result ?: 'SUCCESS'
    def color = status == 'SUCCESS' ? '00FF00' : 'FF0000'
    def title = status == 'SUCCESS' ? 'Build Succeeded' : "Build Failed at stage: ${stageName}"

    writeFile file: 'payload.json', text: """{
      "@type": "MessageCard",
      "@context": "http://schema.org/extensions",
      "summary": "${title}",
      "themeColor": "${color}",
      "title": "${title}: ${env.JOB_NAME}",
      "text": "**Build number: ${env.BUILD_NUMBER}** <br>**URL: [${env.BUILD_URL}](${env.BUILD_URL})**"
    }"""

    bat """
        curl -H "Content-Type: application/json" -d @payload.json "${setNoti}"
    """

    bat "if exist payload.json del payload.json"
}
