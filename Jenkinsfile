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
        string(name: 'JENKINS_AGENT_LABEL',defaultValue:'oc49 || oc48 || oc47')
        choice(name: 'TEST_PRESET', choices: ['NONE', 'SINGLE_POD_2K', 'SINGLE_POD_2500', 'SINGLE_NODE_MULTI_POD_2500', 'MULTI_NODE_10K'], description: 'Preset test cases. Overrides NUM_PROJECTS, RATE, and NUM_LINES')
        string(name: 'NUM_PROJECTS', defaultValue:'1', description:'Number of logtest projects to create')
        string(name: 'RATE', defaultValue:'120000', description:'Number of logs to generate per minute per project. Default: 2k/s')
        string(name: 'NUM_LINES', defaultValue:'3600000', description:'Number of logs to generate before generator exits. Default: 3.6 million for default 30 minute test')
        string(name: 'PROJECT_BASENAME', defaultValue:'logtest', description:'Project name prefix')
        string(name: 'LABEL_NODES_INSTANCETYPE', defaultValue:'m6i.xlarge', description:'Instance type to label with placement=logtest. This label indicates which nodes will have logtest pods assigned to them. Leave blank to skip this step. Note: labels ALL non master nodes that match the instance type')
        string(name: 'QUERY_PATH', defaultValue:'kubernetes.namespace_name', description:'This is passed into the elasticsearch query. Schema is kubernetes.pod_namespace in vector. Should be fixed in GA.')
        // Cleanup
        booleanParam(name: 'CLEANUP_LOGGING', defaultValue: true, description: 'Delete logtest projects, elasticsearch app index, and clear fluentd buffers')
        string(name: 'WORKLOADS_REPO', defaultValue:'https://github.com/SachinNinganure/workloads', description:'You can change this to point to your fork if needed.')
        string(name: 'WORKLOADS_REPO_BRANCH', defaultValue:'logtest_V2L', description:'You can change this to point to a branch on your fork if needed.')
        text(name: 'ENV_VARS', defaultValue: '', description:'''<p>
               Enter list of additional (optional) Env Vars you'd want to pass to the script, one pair on each line. <br>
               e.g.<br>
               SOMEVAR1='env-test'<br>
               SOMEVAR2='env2-test'<br>
               ...<br>
               SOMEVARn='envn-test'<br>
               </p>'''
            )
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
        stage('Checkout workload repo'){
          steps{
            git branch: params.WORKLOADS_REPO_BRANCH, url: params.WORKLOADS_REPO
          }
        }
        stage('Run workload'){
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
              echo PRESET: $TEST_PRESET

              ls -la

              if [ -n $LABEL_NODES_INSTANCETYPE ]; then
                for i in $(oc get machines -n openshift-machine-api -o json | jq -r '.items[] | select(.spec.providerSpec.value.instanceType == env.LABEL_NODES_INSTANCETYPE and .metadata.labels."machine.openshift.io/cluster-api-machine-role" != "master").status.nodeRef.name'); do oc label node --overwrite "$i" placement=logtest; done
              fi

              echo -e "[orchestration]\nlocalhost ansible_connection=local" > inventory
              cat inventory

              if [ "$TEST_PRESET" = "SINGLE_POD_2K" ]; then
                ORCHESTRATION_USER="$(whoami)" LABEL_ALL_NODES=False NUM_PROJECTS=1 NUM_LINES=3600000 RATE=120000 PROJECT_BASENAME=$PROJECT_BASENAME QUERY_PATH=$QUERY_PATH ansible-playbook -v -i inventory workloads/logging.yml -v --skip-tags label_node,clear_buffers,delete_indices
              elif [ "$TEST_PRESET" = "SINGLE_POD_2500" ]; then
                ORCHESTRATION_USER="$(whoami)" LABEL_ALL_NODES=False NUM_PROJECTS=1 NUM_LINES=4500000 RATE=150000 PROJECT_BASENAME=$PROJECT_BASENAME QUERY_PATH=$QUERY_PATH ansible-playbook -v -i inventory workloads/logging.yml -v --skip-tags label_node,clear_buffers,delete_indices
              elif [ "$TEST_PRESET" = "SINGLE_NODE_MULTI_POD_2500" ]; then
                ORCHESTRATION_USER="$(whoami)" LABEL_ALL_NODES=False NUM_PROJECTS=50 NUM_LINES=90000 RATE=3000 PROJECT_BASENAME=$PROJECT_BASENAME QUERY_PATH=$QUERY_PATH ansible-playbook -v -i inventory workloads/logging.yml -v --skip-tags label_node,clear_buffers,delete_indices
              elif [ "$TEST_PRESET" = "MULTI_NODE_10K" ]; then
                ORCHESTRATION_USER="$(whoami)" LABEL_ALL_NODES=False NUM_PROJECTS=10 NUM_LINES=1800000 RATE=60000 PROJECT_BASENAME=$PROJECT_BASENAME QUERY_PATH=$QUERY_PATH ansible-playbook -v -i inventory workloads/logging.yml -v --skip-tags label_node,clear_buffers,delete_indices
              else
                ORCHESTRATION_USER="$(whoami)" LABEL_ALL_NODES=False NUM_PROJECTS=$NUM_PROJECTS NUM_LINES=$NUM_LINES RATE=$RATE PROJECT_BASENAME=$PROJECT_BASENAME QUERY_PATH=$QUERY_PATH ansible-playbook -v -i inventory workloads/logging.yml -v --skip-tags label_node,clear_buffers,delete_indices
              fi
              '''
            }
          }
        }
        stage('Cleanup'){
        when {
          environment name: "CLEANUP_LOGGING", value: 'true'
        }
          steps{
            script {
              build job: 'scale-ci/ematysek-e2e-benchmark/logging-cleanup', parameters: [string(name: 'BUILD_NUMBER', value: "${params.BUILD_NUMBER}"),
              string(name: 'PROJECT_BASENAME', value: "${params.PROJECT_BASENAME}"),
                text(name: 'ENV_VARS', value: "${params.ENV_VARS}"),
                string(name: 'JENKINS_AGENT_LABEL', value: "${params.JENKINS_AGENT_LABEL}")
              ]
            }
          }
        }
      }
    }

  }
}

