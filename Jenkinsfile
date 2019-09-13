#!groovy


// RANCHER_VERSION resolution is first via Jenkins Build Parameter RANCHER_VERSION fed in from console,
// then from $DOCKER_TRIGGER_TAG which is sourced from the Docker Hub Jenkins plugin webhook.
def rancher_version() {
  try { if ('' != RANCHER_VERSION) { return RANCHER_VERSION } }
  catch (MissingPropertyException e) {}

  try { return DOCKER_TRIGGER_TAG }
  catch (MissingPropertyException e) {}

  echo  'Neither RANCHER_VERSION nor DOCKER_TRIGGER_TAG have been specified!'
  error()
}


// Get the AWS prefix if it exists
def aws_prefix() {
  try { if ('' != AWS_PREFIX) { return AWS_PREFIX } }
  catch (MissingPropertyException e) { return false }
}


// Stop the pipeline after provision / deprovision for QA to do something manual
def pipeline_provision_stop() {
  try { if ('' != PIPELINE_PROVISION_STOP) { return PIPELINE_PROVISION_STOP } }
  catch (MissingPropertyException e) { return false }

}

def pipeline_deprovision_stop() {
  try { if ('' != PIPELINE_DEPROVISION_STOP) { return PIPELINE_DEPROVISION_STOP } }
  catch (MissingPropertyException e) { return false }
}


// Run the full validation tests or just a smoke test
def pipeline_smoke_test_only() {
  try { if ('' != SMOKE_TEST_ONLY)  { return PIPELINE_SMOKE_TEST_ONLY } }
  catch (MissingPropertyException e) { return false }
}


// Allow specification of custom validation test command
def custom_validation_test_cmd() {
  try { if ('' != CUSTOM_VALIDATION_TEST_CMD) { return CUSTOM_VALIDATION_TEST_COMMAND } }
  catch (MissingPropertyException e) { return false }
}


// SLACK_CHANNEL resolution is first via Jenkins Build Parameter SLACK_CHANNEL fed in from console,
// then from $DOCKER_TRIGGER_TAG which is sourced from the Docker Hub Jenkins plugin webhook.
def slack_channel() {
  try { if ('' != SLACK_CHANNEL) { return SLACK_CHANNEL } }
  catch (MissingPropertyException e) { return '#ci_cd' }
}


// PIPELINE_POST_SERVER_WAIT will specify duration in seconds to wait on infrastructure catalogs to deploy to Agents
def post_server_wait() {
  try { if ('' != PIPELINE_POST_SERVER_WAIT) { return PIPELINE_POST_SERVER_WAIT } }
  catch (MissingPropertyException e) { return '600' }
}


// simplify the generation of Slack notifications for start and finish of Job
def jenkinsSlack(type) {
  channel = slack_channel()
  aws_prefix = aws_prefix()
  def rancher_version = rancher_version()
  def jobInfo = "\n » ${aws_prefix} ${rancher_version} :: ${JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|job>) (<${env.BUILD_URL}/console|console>)"

  if (type == 'start'){
    slackSend channel: channel, color: 'blue', message: "build started${jobInfo}"
  }
  if (type == 'finish'){
    def buildColor = currentBuild.result == null? "good": "warning"
    def buildStatus = currentBuild.result == null? "SUCCESS": currentBuild.result
    def msg = "build finished - ${buildStatus}${jobInfo}"
    slackSend channel: channel, color: buildColor, message: "${msg}"
  }
}


def lastBuildResult() {
 def previous_build = currentBuild.getPreviousBuild()
  if ( null != previous_build ) { return previous_build.result } else { return 'UKNOWN' }
}


def via_webhook() {
  try {
    def foo = DOCKER_TRIGGER_TAG
    return true
  } catch(MissingPropertyException) {
    return false
  }
}


// validation-tests code needs K8S_DEPLOY set if RANCHER_ORCHESTRATION=k8s
def k8s_deploy() {
  try {
    if ('k8s' == RANCHER_ORCHESTRATION) { return 'True' }
    else { return '' }
  }
  catch (MissingPropertyException e) {}
}


// compute the appropriate validation tests command if the user has not specifically supplied one
def validation_tests_cmd() {

  if ( "true" == "${PIPELINE_SMOKE_TEST_ONLY}" ) {
    return "py.test -s --junit-xml=results.xml validation-tests/tests/v2_validation/cattlevalidationtest/core/test_container_run_option.py"

  } else if ( "k8s" == "${RANCHER_ORCHESTRATION}" ) {
    return "py.test -s --junit-xml=results.xml validation-tests/tests/v2_validation/cattlevalidationtest/core/test_k8*"

  } else {
    return "ONTAG_RUNS='True' py.test -s --junit-xml=results.xml validation-tests/tests/v2_validation/cattlevalidationtest/"
  }
}


// Get filename where CATTLE_TEST_URL is stored **for this build**.
// Format: cattle_test_url.<BUILD_NUMBER>
def cattle_test_url_filename() {
  try {
    return "cattle_test_url.${env.BUILD_NUMBER}"
  } catch (MissingPropertyException e) {
    echo "BUILD_NUMBER is required but was not set in env!"
    error()
  }
}

def project_id_filename() {
  try {
    return "project_id.${env.BUILD_NUMBER}"
  } catch (MissingPropertyException e) {
    echo "BUILD_NUMBER is required but was not set in env!"
    error()
  }
}

// Filter out Docker Hub tags like 'latest', 'master', 'enterprise'.
// Just want things like v1.2*
def rancher_version = rancher_version()
def String rancher_version_regex = "^v[\\d]\\.[\\d]\\.[\\d][\\-rc\\d]+\$"

// To accomodate for patch releases
// def String rancher_version_regex = "^v[\\d]\\.[\\d]\\.[\\d][\\-patch\\d]?[\\-rc\\d]+\$"

if ( true == via_webhook() && (!(rancher_version ==~ rancher_version_regex)) ) {
  println("Received RANCHER_VERSION \'${rancher_version}\' via webhook which does not match regex \'${rancher_version_regex}\'.")
  println("** This will **not** result in a pipeline run.")
  currentBuild.result = lastBuildResult()
} else {
  try {
    node {
      wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm', 'defaultFg': 2, 'defaultBg':1]) {
        withCredentials([ string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                          string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
                          string(credentialsId: 'SLACK_TOKEN', variable: 'SLACK_TOKEN'),
                          string(credentialsId: 'DO_ACCESSKEY_1.6', variable: 'DIGITALOCEAN_KEY')]) {  
          jenkinsSlack('start')
          checkout scm

          stage('bootstrap') {
            sh "./scripts/bootstrap.sh"
          }

          stage ('syntax') {
            sh "docker run --rm  " +
              "-v jenkins_home:/var/jenkins_home " +
              "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke syntax\'"
          }

          stage ('lint') {
            sh "docker run --rm  " +
              "-v jenkins_home:/var/jenkins_home " +
              "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke lint\'"
          }

          stage ('configure .env file') {
            withEnv(["RANCHER_VERSION=${rancher_version}"]) {
              sh "./scripts/configure.sh"
            }
          }

          stage ('deprovision Rancher Agents') {
            sh "docker run --rm  " +
              "-v jenkins_home:/var/jenkins_home " +
              "--env-file .env " +
              "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_agents.deprovision\'"
          }

          stage ('deprovision rancher/server') {
            sh "docker run --rm  " +
              "-v jenkins_home:/var/jenkins_home " +
              "--env-file .env " +
              "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_server.deprovision\'"
          }

          if ( "false" == "${PIPELINE_DEPROVISION_STOP}" ) {

          // stage ('provision AWS') {
          //   sh "docker run --rm  " +
          //     "-v jenkins_home:/var/jenkins_home " +
          //     "--env-file .env " +
          //     "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke aws.provision\'"
          // }

            stage('provision rancher/server') {
              sh "docker run --rm  " +
                "-v jenkins_home:/var/jenkins_home " +
                "--env-file .env " +
                "-e WORKSPACE_DIR=\"\$(pwd)\" " +
                "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_server.provision\'"
            }

            stage ('configure rancher/server') {
              sh "docker run --rm  " +
                "-v jenkins_home:/var/jenkins_home " +
                "--env-file .env " +
                "-e WORKSPACE_DIR=\"\$(pwd)\" " +
                "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_server.configure\'"
            }

            stage ('provision Rancher Agents') {
              sh "docker run --rm  " +
          "-v jenkins_home:/var/jenkins_home " +
          "--env-file .env " +
          "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_agents.provision\'"
            }

            if ( "false" == "${PIPELINE_PROVISION_STOP}" ) {
              stage ('wait for infra catalogs to settle...') {
                post_server_wait = post_server_wait()
                withEnv(["PIPELINE_POST_SERVER_WAIT=${post_server_wait}"]) {
                  sh "echo 'Sleeping for ${PIPELINE_POST_SERVER_WAIT} seconds while we wait on infrastructure catalogs to deploy to Agents....'"
                  sh "sleep ${PIPELINE_POST_SERVER_WAIT}"
                }
              }

              stage ('run validation tests') {
                CATTLE_TEST_URL = readFile(cattle_test_url_filename()).trim()
                PROJECT_ID = readFile(project_id_filename()).trim()

                withEnv(["CATTLE_TEST_URL=${CATTLE_TEST_URL}", "PROJECT_ID=${PROJECT_ID}", "ACCESS_KEY=test", "SECRET_KEY=test"]) {
                  sh "git clone https://github.com/rancher/validation-tests"
                  try {
                    def cmd = validation_tests_cmd()
                    sh "${cmd}"
                  } catch(err) {
                    echo 'Test run had failures. Collecting results...'
                  }
                }

                step([$class: 'JUnitResultArchiver', testResults: '**/results.xml'])
              }

              stage ('deprovision Rancher Agents') {
                sh "docker run --rm  " +
            "-v jenkins_home:/var/jenkins_home " +
            "--env-file .env " +
            "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_agents.deprovision\'"
              }

              stage ('deprovision rancher/server') {
                sh "docker run --rm  " +
            "-v jenkins_home:/var/jenkins_home " +
            "--env-file .env " +
            "rancherlabs/ci-validation-tests /bin/bash -c \'cd \"\$(pwd)\" && invoke rancher_server.deprovision\'"
              }
            } // PIPELINE_PROVISION_STOP
          } // PIPELINE_DEPROVISION_STOP
        } // creds
      } // wrap
    } // node
  } catch(err) { currentBuild.result = 'FAILURE' }
}

jenkinsSlack('finish')
