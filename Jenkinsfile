@Library('flexy') _

// rename build
def userId = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause)?.userId
if (userId) {
  currentBuild.displayName = userId
}

pipeline {
  agent none

  parameters {
        string(name: 'BUILD_NUMBER', defaultValue: '', description: 'Build number of job that has installed the cluster.')
        booleanParam(name: 'SCALE_MACHINESETS', defaultValue: true, description: 'Scale worker machinesets to 0, update instance type as specified, then scale up. machineset A is scaled for elasticsearch, B is scaled for fluentd, and all other machinesets are scaled down to 0')
        string(name: 'ELS_INSTANCE_TYPE', defaultValue: 'm6i.2xlarge', description: '')
        string(name: 'COLLECTOR_INSTANCE_TYPE', defaultValue: 'm6i.xlarge', description: '')
        string(name: 'NUM_LOGTEST_NODES', defaultValue: '1', description: '')
        booleanParam(name: 'DEPLOY_LOGGING', defaultValue: true, description: 'Deploy cluster loging operator and elasticsearch operator')
        string(name: 'CLO_BRANCH', defaultValue: 'release-5.3', description: 'Branch to deploy Cluster Logging Operator and Elasticsearch Operator from. See https://github.com/openshift/cluster-logging-operator')
        booleanParam(name: 'CREATE_CLO_INSTANCE', defaultValue: true, description: 'Create CLO instance with ELS 4 cpu, 16G RAM, and 200G gp2 ssd storage.')
        choice(name: 'LOG_COLLECTOR', choices: ['fluentd', 'vector'], description: 'Log collector to use.')
        string(name: 'SLEEP_DELAY', defaultValue: '10m', description: 'Time to sleep after previous steps to wait for nodes/CLO to be ready. Leave empty to skip')
        string(name: 'JENKINS_AGENT_LABEL',defaultValue:'oc49 || oc48 || oc47')
        string(name: 'LOGGING_HELPER_REPO', defaultValue:'https://github.com/SachinNinganure/openshift-logtest-helper', description:'You can change this to point to your fork if needed.')
        string(name: 'LOGGING_HELPER_REPO_BRANCH', defaultValue:'sn-forLokistack', description:'You can change this to point to a branch on your fork if needed.')
        text(name: 'ENV_VARS', defaultValue: '', description:'''<p>
               Enter list of additional (optional) Env Vars you'd want to pass to the script, one pair on each line. <br>
               e.g.<br>
               SOMEVAR1='env-test'<br>
               SOMEVAR2='env2-test'<br>
               ...<br>
               SOMEVARn='envn-test'<br>
               </p>'''
        )
        booleanParam(name: 'DEBUG', defaultValue: false)
    }

  stages {
    stage('Sequential'){
      agent { label params['JENKINS_AGENT_LABEL'] }
      stages{
        stage('Copy artifacts'){
          steps{
            copyArtifacts(
                filter: '',
                fingerprintArtifacts: true,
                projectName: 'ocp-common/Flexy-install',
                selector: specific(params.BUILD_NUMBER),
                target: 'flexy-artifacts'
            )
            script {
              buildinfo = readYaml file: "flexy-artifacts/BUILDINFO.yml"
              currentBuild.displayName = "${currentBuild.displayName}-${params.BUILD_NUMBER}-${currentBuild.number}"
              currentBuild.description = "Copying Artifact from Flexy-install build <a href=\"${buildinfo.buildUrl}\">Flexy-install#${params.BUILD_NUMBER}</a>"
              buildinfo.params.each { env.setProperty(it.key, it.value) }
            }
          }
        }
        stage('Checkout repo'){
          steps{
            dir('openshift-logtest-helper'){
              git branch: params.LOGGING_HELPER_REPO_BRANCH, url: params.LOGGING_HELPER_REPO
            }
          }
        }
        stage('Debug info'){
          when {
            environment name: 'DEBUG', value: 'true'
          }
          steps{
            ansiColor('xterm') {
              sh label: '', script: '''
              # Get ENV VARS Supplied by the user to this job and store in .env_override
              echo "$ENV_VARS" > .env_override
              # Export those env vars so they could be used by CI Job
              set -a && source .env_override && set +a
              mkdir -p ~/.kube
              cp $WORKSPACE/flexy-artifacts/workdir/install-dir/auth/kubeconfig ~/.kube/config
              #oc config view
              #oc projects
              ls -ls ~/.kube/
              env
              oc version
              oc project default
              ansible --version
              python --version
              python3 --version
              whoami

              ls -la
              ls openshift-logtest-helper


              '''
            }
          }
        }
        stage('Scale Machinesets'){
          when {
            environment name: 'SCALE_MACHINESETS', value: 'true'
          }
          steps{
            ansiColor('xterm') {
              sh label: '', script: '''
              # Get ENV VARS Supplied by the user to this job and store in .env_override
              echo "$ENV_VARS" > .env_override
              # Export those env vars so they could be used by CI Job
              set -a && source .env_override && set +a
              mkdir -p ~/.kube
              cp $WORKSPACE/flexy-artifacts/workdir/install-dir/auth/kubeconfig ~/.kube/config
              ls -la
              cd openshift-logtest-helper
              ./prep_machinesets.sh
              '''
            }
          }
        }
        stage('Deploy Lokistack and Cluster logging'){
          when {
            environment name: 'DEPLOY_LOGGING', value: 'true'
          }
          steps{
            ansiColor('xterm') {
              sh label: '', script: '''
              # Get ENV VARS Supplied by the user to this job and store in .env_override
              echo "$ENV_VARS" > .env_override
              # Export those env vars so they could be used by CI Job
              set -a && source .env_override && set +a
              mkdir -p ~/.kube
              cp $WORKSPACE/flexy-artifacts/workdir/install-dir/auth/kubeconfig ~/.kube/config
              ls -la
              cd openshift-logtest-helper
              ./deploy_logging.sh 
              '''
            }
          }
        }
        stage('Sleep'){
          when {
            not { environment name: "SLEEP_DELAY", value: "" }
          }
          steps{
            ansiColor('xterm') {
              sh label: '', script: '''
                sleep "$SLEEP_DELAY"
              '''
            }
          }
        }
      }
    }

  }
}

