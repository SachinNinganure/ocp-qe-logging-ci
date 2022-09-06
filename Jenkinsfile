@Library('flexy') _

// rename build
def userId = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause)?.userId
if (userId) {
  currentBuild.displayName = userId
}

pipeline {
  agent none

  parameters {
        // Build cluster
        string(name: 'BUILD_NUMBER', defaultValue: '---', description: 'Leave blank if building cluster as part of e2e. Build number of job that has installed the cluster.')
        string(name: 'INSTANCE_NAME_PREFIX', defaultValue: 'e2elogging', description: '')
        string(name: 'VARIABLES_LOCATION', defaultValue: 'private-templates/functionality-testing/aos-4_10/ipi-on-aws/versioned-installer', description: '')
        string(name: 'INSTALLER_PAYLOAD_IMAGE', defaultValue: 'registry.ci.openshift.org/ocp/release:4.10.6', description: '')
        // Prep cluster
        booleanParam(name: 'SCALE_MACHINESETS', defaultValue: true, description: 'Scale worker machinesets to 0, update instance type as specified, then scale up. machineset A is scaled for elasticsearch, B is scaled for Vector, and all other machinesets are scaled down to 0')
        string(name: 'ELS_INSTANCE_TYPE', defaultValue: 'm6i.2xlarge', description: '')
        string(name: 'COLLECTOR_INSTANCE_TYPE', defaultValue: 'm6i.xlarge', description: '')
        string(name: 'NUM_LOGTEST_NODES', defaultValue: '5', description: '')
        string(name: 'SLEEP_DELAY', defaultValue: "10m", description: 'Time to sleep after previous steps to wait for nodes/CLO to be ready. Leave empty to skip')
        // Workload
        choice(name: 'TEST_PRESET', choices: ['NONE', 'SINGLE_POD_2K', 'SINGLE_POD_2500', 'SINGLE_NODE_MULTI_POD_2500', 'MULTI_NODE_10K'], description: 'Preset test cases. Overrides NUM_PROJECTS, RATE, and NUM_LINES')
        string(name: 'PROJECT_BASENAME', defaultValue:'logtest', description:'Project name prefix')
        string(name: 'LABEL_NODES_INSTANCETYPE', defaultValue:'m6i.xlarge', description:'Instance type to label with placement=logtest. This label indicates which nodes will have logtest pods assigned to them. Leave blank to skip this step. Note: labels ALL non master nodes that match the instance type')
        string(name: 'QUERY_PATH', defaultValue:'kubernetes.namespace_name', description:'This is passed into the elasticsearch query. Schema is kubernetes.pod_namespace in vector. Should be fixed in GA.')
        // Cleanup
        booleanParam(name: 'CLEANUP_LOGGING', defaultValue: true, description: 'Delete logtest projects, elasticsearch app index, and clear fluentd buffers')

        // Global
        string(name: 'JENKINS_AGENT_LABEL',defaultValue:'oc49 || oc48 || oc47')
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
        stage('Build Cluster'){
          steps {
            script {
              buildno = ""
              if(params.BUILD_NUMBER == "") {
                install = build job:"ocp-common/Flexy-install", parameters: [string(name: 'INSTANCE_NAME_PREFIX', value: "${params.INSTANCE_NAME_PREFIX}"),
                  string(name: 'VARIABLES_LOCATION', value: "${params.VARIABLES_LOCATION}"),
                  text(name: 'LAUNCHER_VARS', value: "installer_payload_image: ${params.INSTALLER_PAYLOAD_IMAGE}\nuse_internal_opsrc: \"yes\"\ninstall_logging: \"yes\""),
                  text(name: 'BUSHSLICER_CONFIG', value: '''services:
  AWS-CI:
    config_opts:
      region: us-east-2'''),
                  string(name: 'JENKINS_AGENT_LABEL', value: "${params.JENKINS_AGENT_LABEL}")
                ]
                buildno = install.number.toString()
              } else {
                buildno = params.BUILD_NUMBER
              }
            }
          }
        }
        stage('Prep Cluster'){
          when {
            anyOf {
              environment name: "SCALE_MACHINESETS", value: 'true'
              environment name: "DEPLOY_LOGGING", value: 'true'
              environment name: "CREATE_CLO_INSTANCE", value: 'true'
            }
          }
          steps {
            script {
              build job: 'scale-ci/ematysek-e2e-benchmark/logging-prep-cluster', parameters: [string(name: 'BUILD_NUMBER', value: "${buildno}"),
                booleanParam(name: 'SCALE_MACHINESETS', value: "${params.SCALE_MACHINESETS}"),
                string(name: 'ELS_INSTANCE_TYPE', value: "${params.ELS_INSTANCE_TYPE}"),
                string(name: 'COLLECTOR_INSTANCE_TYPE', value: "${params.COLLECTOR_INSTANCE_TYPE}"),
                string(name: 'NUM_LOGTEST_NODES', value: "${params.NUM_LOGTEST_NODES}"),
                string(name: 'SLEEP_DELAY', value: "${params.SLEEP_DELAY}"),
                text(name: 'ENV_VARS', value: "${params.ENV_VARS}"),
                string(name: 'JENKINS_AGENT_LABEL', value: "${params.JENKINS_AGENT_LABEL}")
              ]
            }
          }
        }
        stage('Run Workload'){
          steps {
            script {
              build job: 'scale-ci/ematysek-e2e-benchmark/V2L_Logging', parameters: [string(name: 'BUILD_NUMBER', value: "${buildno}"),
                string(name: 'TEST_PRESET', value: "${params.TEST_PRESET}"),
                string(name: 'PROJECT_BASENAME', value: "${params.PROJECT_BASENAME}"),
                string(name: 'LABEL_NODES_INSTANCETYPE', value: "${params.LABEL_NODES_INSTANCETYPE}"),
                string(name: 'QUERY_PATH', value: "${params.QUERY_PATH}"),
                booleanParam(name: 'CLEANUP_LOGGING', value: 'false'),
                text(name: 'ENV_VARS', value: "${params.ENV_VARS}"),
                string(name: 'JENKINS_AGENT_LABEL', value: "${params.JENKINS_AGENT_LABEL}")
              ]
            }
          }
        }
        stage('Cleanup'){
        when {
          environment name: "CLEANUP_LOGGING", value: 'true'
        }
          steps{
            script {
              build job: 'scale-ci/ematysek-e2e-benchmark/logging-cleanup', parameters: [string(name: 'BUILD_NUMBER', value: "${buildno}"),
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

