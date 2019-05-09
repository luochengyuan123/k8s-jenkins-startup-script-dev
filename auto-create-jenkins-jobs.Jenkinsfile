def isJobExist(String jobName) {
    def response = httpRequest(
            authentication: '9b041abc-e0e6-43ec-a87c-f65a74a125c8',
            httpMode: 'GET',
            ignoreSslErrors: true,
            validResponseCodes: '200:500',
            url: "http://jenkins.t.payeco.com/job/${jobName}/description")
    if (response.status == 200) {
        return true
    }
    return false
}

def createJob(String projectName, String jobName, String jobDesc, List envTag, String branchFilter) {
    def confgFile = """<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@2.31">
    <actions/>
    <description/>
    <displayName>[${envTag[0]}] ${jobDesc} ${projectName}</displayName>
    <keepDependencies>false</keepDependencies>
    <properties>
        <hudson.model.ParametersDefinitionProperty>
            <parameterDefinitions>
                <net.uaznia.lukanus.hudson.plugins.gitparameter.GitParameterDefinition plugin="git-parameter@0.9.10">
                    <name>BRANCH</name>
                    <description/>
                    <uuid>01241d25-ae55-4ec0-a39c-3fc3a0342a48</uuid>
                    <type>PT_BRANCH</type>
                    <branch/>
                    <tagFilter>*</tagFilter>
                    <branchFilter>origin/(${branchFilter}.*)</branchFilter>
                    <sortMode>DESCENDING_SMART</sortMode>
                    <defaultValue>master</defaultValue>
                    <selectedValue>NONE</selectedValue>
                    <useRepository>http://10.123.107.106/payeco/${projectName}.git</useRepository>
                    <quickFilterEnabled>false</quickFilterEnabled>
                    <listSize>5</listSize>
                </net.uaznia.lukanus.hudson.plugins.gitparameter.GitParameterDefinition>
            </parameterDefinitions>
        </hudson.model.ParametersDefinitionProperty>
    </properties>
    <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition" plugin="workflow-cps@2.63">
        <scm class="hudson.plugins.git.GitSCM" plugin="git@3.9.3">
            <configVersion>2</configVersion>
            <userRemoteConfigs>
                <hudson.plugins.git.UserRemoteConfig>
                    <url>http://10.123.107.106/payeco/k8s-jenkins-startup-script-dev.git</url>
                    <credentialsId>692a8f27-c716-42c8-a334-81db86d891a5</credentialsId>
                </hudson.plugins.git.UserRemoteConfig>
            </userRemoteConfigs>
            <branches>
                <hudson.plugins.git.BranchSpec>
                    <name>*/master</name>
                </hudson.plugins.git.BranchSpec>
            </branches>
            <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
            <submoduleCfg class="list"/>
            <extensions/>
        </scm>
        <scriptPath>deploy.Jenkinsfile</scriptPath>
        <lightweight>true</lightweight>
    </definition>
    <triggers/>
    <disabled>false</disabled>
</flow-definition>
"""
    writeFile encoding: 'UTF-8', file: "config.xml", text: confgFile
    withCredentials([usernamePassword(credentialsId: "9b041abc-e0e6-43ec-a87c-f65a74a125c8", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        sh "curl -u ${USERNAME}:${PASSWORD} -X POST http://jenkins.t.payeco.com/createItem?name=${jobName} --data-binary @config.xml -H 'Content-Type: text/xml'"
    }
}

node('jenkins-master') {
    def platEnv = params.Environment.trim()
    def totalPages = sh(script: "curl -I -s  \"http://10.123.107.106/api/v3/projects?simple=true&private_token=ET61-7jU2do6g18pUemx\" | awk -F: '/x-total-pages/{print \$2}'", returnStdout: true).trim()
    println("TotalPages: " + "${totalPages}")
    for (int page = 1; page <= Integer.parseInt(totalPages); page++) {
        def response = httpRequest "http://10.123.107.106/api/v3/projects?simple=true&private_token=ET61-7jU2do6g18pUemx&&page=${page}"
        def projects = readJSON text: response.content
        def envTag = ""
        def jobName = ""
        def branchFilter = ""
        for (i = 0; i < projects.size(); i++) {
            if (platEnv == "Test") {
                jobName = "t-" + "${projects[i].name}"
                envTag = ["T", "test"]
                branchFilter = "[release|hotfix]"
            } else if (platEnv == "Dev") {
                jobName = "d-" + "${projects[i].name}"
                envTag = ["D", "dev"]
            } else if ( platEnv == "Joint") {
                jobName = "j-" + "${projects[i].name}"
                envTag = ["J", "joint"]
                branchFilter = "[release|hotfix]"
            }
            println("jobName: " + "${jobName}")
            for (tag in projects[i].tag_list) {
                if (tag == "springboot") {
                    if (!isJobExist(jobName)) {
                        println("Createing Job: " + "${jobName}")
                        dir("/var/jenkins_home/workspace/tmp") {
                            createJob(projects[i].name, jobName, projects[i].description, envTag, branchFilter)
                        }
                    } else {
                        println("Job Already Exist:: " + "${jobName}")
                    }
                }
            }
        }
    }
}