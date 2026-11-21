
pipeline {
    agent any

    parameters {
        string(
            name: 'JIRA_ISSUE',
            defaultValue: 'LIA-2',
            description: 'Enter Jira ticket number'
        )
    }

    options {
        disableConcurrentBuilds()
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        ANSIBLE_FORCE_COLOR = 'True'

        ANSIBLE = '/opt/ansible-venv/bin/ansible'
        ANSIBLE_PLAYBOOK = '/opt/ansible-venv/bin/ansible-playbook'

        INVENTORY = 'inventory.ini'
        PEM_KEY = '/var/lib/jenkins/workspace/Terraform/rhel-terraform-key.pem'

        JIRA_API_BASE = 'https://api.atlassian.com/ex/jira/88cf78c6-5fcd-41ea-b0cf-e3ab55e51a1d'
    }

    stages {

        stage('Checkout Ansible Repository') {
            steps {
                checkout scm
            }
        }

        stage('Check Tools and Files') {
            steps {
                sh '''
                    set -e

                    echo "===== ANSIBLE VERSION ====="
                    $ANSIBLE --version

                    echo "===== CHECK PLAYBOOKS ====="
                    test -f inventory.ini
                    test -f users.yml
                    test -f filesystem.yml
                    test -f swap.yml
                    test -f agents.yml
                    test -f cloudwatch-agent.yml

                    echo "===== CHECK PEM KEY ====="
                    test -r "$PEM_KEY"

                    echo "===== CHECK JIRA TOOLS ====="
                    command -v curl
                    command -v jq

                    echo "All required files are available."
                '''
            }
        }

        stage('Ansible Connectivity Test') {
            steps {
                sh '''
                    set -e

                    echo "Testing SSH connectivity to both RHEL servers"

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -m ping
                '''
            }
        }

        stage('Create Linux Users') {
            steps {
                sh '''
                    set -e

                    $ANSIBLE_PLAYBOOK \
                    -i "$INVENTORY" \
                    users.yml
                '''
            }
        }

        stage('Configure Filesystems and Folders') {
            steps {
                sh '''
                    set -e

                    $ANSIBLE_PLAYBOOK \
                    -i "$INVENTORY" \
                    filesystem.yml
                '''
            }
        }

        stage('Configure Swap') {
            steps {
                sh '''
                    set -e

                    $ANSIBLE_PLAYBOOK \
                    -i "$INVENTORY" \
                    swap.yml
                '''
            }
        }

        stage('Install Required Agents') {
            steps {
                sh '''
                    set -e

                    $ANSIBLE_PLAYBOOK \
                    -i "$INVENTORY" \
                    agents.yml
                '''
            }
        }

        stage('Install and Configure CloudWatch Agent') {
            steps {
                sh '''
                    set -e

                    echo "Installing CloudWatch Agent on both RHEL servers"

                    $ANSIBLE_PLAYBOOK \
                    -i "$INVENTORY" \
                    cloudwatch-agent.yml
                '''
            }
        }

        stage('Verify CloudWatch Agent') {
            steps {
                sh '''
                    set -e

                    echo "Checking CloudWatch Agent service status"

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -b \
                    -m command \
                    -a "systemctl is-active amazon-cloudwatch-agent"

                    echo "CloudWatch Agent verification completed."
                '''
            }
        }

        stage('Final Server Verification') {
            steps {
                sh '''
                    set -e

                    echo "===== SERVER HOSTNAMES ====="

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -m command \
                    -a "hostname"

                    echo "===== ROOT FILESYSTEM USAGE ====="

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -m command \
                    -a "df -h /"

                    echo "===== ALL FILESYSTEMS ====="

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -b \
                    -m command \
                    -a "df -h"

                    echo "Final server verification completed."
                '''
            }
        }

        stage('Update Jira Deployment Details') {
            steps {
                catchError(
                    buildResult: 'UNSTABLE',
                    stageResult: 'UNSTABLE',
                    message: 'Infrastructure configuration succeeded, but Jira reporting failed.'
                ) {
                    script {
                        def issueKey = params.JIRA_ISSUE?.trim()

                        if (!issueKey) {
                            error('JIRA_ISSUE parameter is empty.')
                        }

                        if (!(issueKey ==~ /[A-Z][A-Z0-9_]*-[0-9]+/)) {
                            error('Invalid Jira issue key: ' + issueKey)
                        }

                        withEnv(["JIRA_ISSUE=${issueKey}"]) {
                            withCredentials([
                                usernamePassword(
                                    credentialsId: 'jira-cloud-api',
                                    usernameVariable: 'JIRA_EMAIL',
                                    passwordVariable: 'JIRA_API_TOKEN'
                                )
                            ]) {
                                sh '''
                                    set +x
                                    set -eu

                                    echo "===== JIRA DEPLOYMENT UPDATE ====="
                                    echo "Jira Issue: $JIRA_ISSUE"

                                    MESSAGE=$(cat <<EOF
Infrastructure configuration pipeline completed successfully.

Jenkins Job: ${JOB_NAME}
Build Number: ${BUILD_NUMBER}
Build URL: ${BUILD_URL}

Configured Operating System: RHEL 9.8
Automation Tools: Jenkins and Ansible

Completed Activities:
- Ansible connectivity verification
- Linux user and group configuration
- Filesystem and folder configuration
- Swap configuration
- Required Telegraf monitoring agent installation
- CloudWatch Agent configuration
- CloudWatch Agent service verification
- Final server verification

Infrastructure provisioning:
Existing AWS EC2 virtual machines.

Terraform provisioning:
Not executed by this pipeline run.

Splunk installation:
Performed separately.

Splunk forwarding:
Not validated by this pipeline run.

Pipeline Result: SUCCESS
EOF
)

                                    PAYLOAD=$(jq -n \
                                      --arg message "$MESSAGE" \
                                      '{
                                        body: {
                                          type: "doc",
                                          version: 1,
                                          content: [{
                                            type: "paragraph",
                                            content: [{
                                              type: "text",
                                              text: $message
                                            }]
                                          }]
                                        }
                                      }')

                                    RESPONSE_FILE=$(mktemp)
                                    trap 'rm -f "$RESPONSE_FILE"' EXIT

                                    HTTP_STATUS=$(curl \
                                      --connect-timeout 15 \
                                      --max-time 45 \
                                      -sS \
                                      -o "$RESPONSE_FILE" \
                                      -w '%{http_code}' \
                                      -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
                                      -X POST \
                                      -H "Accept: application/json" \
                                      -H "Content-Type: application/json" \
                                      "$JIRA_API_BASE/rest/api/3/issue/$JIRA_ISSUE/comment" \
                                      --data "$PAYLOAD")

                                    echo "Jira HTTP Status: $HTTP_STATUS"

                                    if [ "$HTTP_STATUS" != "201" ]; then
                                        echo "Jira comment update failed:"
                                        cat "$RESPONSE_FILE"
                                        exit 1
                                    fi

                                    COMMENT_ID=$(jq -r '.id // empty' "$RESPONSE_FILE")

                                    echo "Jira comment created successfully."
                                    echo "Jira Comment ID: $COMMENT_ID"
                                '''
                            }
                        }
                    }
                }
            }
        }
    }

    post {

        success {
            echo '''
            =========================================
            ANSIBLE PIPELINE COMPLETED SUCCESSFULLY
            =========================================
            Linux users configured
            Filesystems configured
            Swap configured
            Telegraf agent verified
            CloudWatch Agent verified
            Both RHEL servers checked
            Jira deployment details updated
            =========================================
            '''
        }

        unstable {
            echo '''
            =========================================
            PIPELINE COMPLETED WITH WARNING
            =========================================
            Check the Jira reporting stage.
            Infrastructure configuration stages
            may have completed successfully.
            =========================================
            '''
        }

        failure {
            echo '''
            =========================================
            ANSIBLE PIPELINE FAILED
            =========================================
            Please check the failed stage in Console Output.
            =========================================
            '''
        }

        always {
            echo 'Ansible pipeline execution finished.'
        }
    }
}

