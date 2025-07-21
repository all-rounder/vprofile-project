def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
pipeline {
    agent any
    tools {
        maven "MAVEN3.9.9"
        jdk "JDK21"
    }
    
    environment {
        NEXUSIP = '172.31.7.72'
		NEXUSPORT = '8081'
		NEXUSUSER = 'admin'
        NEXUSPASS = credentials('nexuspass')
        NEXUS-GRP-REPO = 'vpro-maven-group'
        CENTRAL-REPO = 'vpro-maven-central'
		RELEASE-REPO = 'vprofile-release'
		SNAP-REPO = 'vprofile-snapshot'
        NEXUS-LOGIN = 'nexuslogin'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Build'){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test'){
            steps {
                sh 'mvn -s settings.xml test'
            }

        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage("UploadArtifact"){
            steps{
                nexusArtifactUploader(
                  nexusVersion: 'nexus3',
                  protocol: 'http',
                  nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                  groupId: 'QA',
                  version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                  repository: "${RELEASE-REPO}",
                  credentialsId: "${NEXUS-LOGIN}",
                  artifacts: [
                    [artifactId: 'vproapp',
                     classifier: '',
                     file: 'target/vprofile-v2.war',
                     type: 'war']
                  ]
                )
            }
        }

        stage('Ansible Deploy to staging'){
            steps {
                ansiblePlaybook([
                inventory   : 'ansible/stage.inventory',
                playbook    : 'ansible/site.yml',
                installation: 'ansible',
                colorized   : true,
			    credentialsId: 'applogin',
			    disableHostKeyChecking: true,
                extraVars   : [
                   	USER: "${NEXUSUSER}",
                    PASS: "${NEXUSPASS}",
			        nexusip: "172.31.7.72",
			        reponame: "vprofile-release",
			        groupid: "QA",
			        time: "${env.BUILD_TIMESTAMP}",
			        build: "${env.BUILD_ID}",
                    artifactid: "vproapp",
			        vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                ]
             ])
            }
        }

    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}