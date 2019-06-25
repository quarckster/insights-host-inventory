/*
 * Requires: https://github.com/RedHatInsights/insights-pipeline-lib
 */

@Library("github.com/RedHatInsights/insights-pipeline-lib") _

// Name for auto-generated openshift pod
podLabel = "host-inventory-test-${UUID.randomUUID().toString()}"

// Code coverage failure threshold
codecovThreshold = 80

venvDir = "venv"

node {
    cancelPriorBuilds()

    runIfMasterOrPullReq {
        runStages()
    }
}

def get_sonar_cli(){
        sh """
            curl --insecure -o ./sonarscanner.zip -L https://repo1.maven.org/maven2/org/sonarsource/scanner/cli/sonar-scanner-cli/3.3.0.1492/sonar-scanner-cli-3.3.0.1492.zip && \
	        unzip sonarscanner.zip && \
	        rm sonarscanner.zip && \
	        mv sonar-scanner-3.3.0.1492 sonar-scanner
        """
}

def sonar_scanner(){
    stage('sonar scanner'){
        get_sonar_cli()
        withCredentials([
            string(credentialsId: 'jenkins-sonarqube', variable: 'TOKEN'),
            string(credentialsId: 'sonarqube-url-dev', variable: 'SONAR_URL')]) {
            sh 'sonar-scanner/bin/sonar-scanner ' +
            '-Dsonar.host.url=${SONAR_URL} ' +
            '-Dsonar.projectKey=insights-host-inventory ' +
            '-Dsonar.language=py ' +
            '-Dsonar.sources=. ' +
            '-Dsonar.projectDescription=Host inventory project ' +
            '-Dsonar.links.ci=https://jenkins-jenkins.5a9f.insights-dev.openshiftapps.com/view/QE/job/qe/job/CI-inventory-sonarqube/view/change-requests/ ' +
            '-Dsonar.links.issue=https://projects.engineering.redhat.com/secure/RapidBoard.jspa?rapidView=2784&projectKey=RHCLOUD ' +
            '-Dsonar.links.scm=https://github.com/RedHatInsights/insights-host-inventory ' +
            '-Dsonar.sourceEncoding=UTF-8 ' +
            '-Dsonar.python.xunit.reportPath=junit.xml ' +
            '-Dsonar.python.coverage.reportPath=coverage.xml ' +
            '-Dsonar.exclusions=**/.pytest_cache/*, ' +
            '-Dsonar.coverage.exclusions=**/tests/*,**/tests/app/*,**/tests/fixtures/*,**/docker/consumer/*,**/tests/fixtures/*,**/sonar-scanner/* ' +
            '-Dsonar.login=${TOKEN} ' +
            '-Dsonar.password= '
        }
    }
}

def runStages() {

    // Fire up a pod on openshift with containers for the DB and the app
    podTemplate(label: podLabel, slaveConnectTimeout: 120, cloud: 'openshift', containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'docker-registry.default.svc:5000/jenkins/jenkins-slave-base-centos7-python36',
            args: '${computer.jnlpmac} ${computer.name}',
            resourceRequestCpu: '200m',
            resourceLimitCpu: '500m',
            resourceRequestMemory: '256Mi',
            resourceLimitMemory: '650Mi'
        ),
        containerTemplate(
            name: 'postgres',
            image: 'postgres:9.6',
            ttyEnabled: true,
            envVars: [
                containerEnvVar(key: 'POSTGRES_USER', value: 'insights'),
                containerEnvVar(key: 'POSTGRES_PASSWORD', value: 'insights'),
                containerEnvVar(key: 'POSTGRES_DB', value: 'insights'),
                containerEnvVar(key: 'PGDATA', value: '/var/lib/postgresql/data/pgdata')
            ],
            volumes: [emptyDirVolume(mountPath: '/var/lib/postgresql/data/pgdata')],
            resourceRequestCpu: '200m',
            resourceLimitCpu: '200m',
            resourceRequestMemory: '100Mi',
            resourceLimitMemory: '100Mi'
        )
    ]) {

        node(podLabel) {
          
            // check out source again to get it in this node's workspace
            scmVars = checkout scm

            stage('Setting up virtual environment') {
                runPipenvInstall(scmVars: scmVars)
            }

            stage('Start app') {
                sh """
                    ${pipelineVars.userPath}/pipenv run python ./manage.py db upgrade
                    ${pipelineVars.userPath}/pipenv run python ./run.py > app.log 2>&1 &
                """
            }

            stage('Lint') {
                runPythonLintCheck()
            }

            stage('Unit tests') {
                withStatusContext.unitTest {
                    sh "${pipelineVars.userPath}/pipenv run pytest --cov=. --junitxml=junit.xml --cov-report xml -s -v"
                    junit 'junit.xml'
                    archiveArtifacts "*.xml"
                }
            }

            stage('Code coverage') {
                checkCoverage(threshold: codecovThreshold)
            }

            archiveArtifacts "app.log"
            archiveArtifacts "README.md"

            sonar_scanner()
        }    
    }
}
