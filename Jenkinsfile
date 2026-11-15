
pipeline {
    agent any

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
        PEM_KEY = '/opt/automation/Terraform/rhel-terraform-key.pem'
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

                    echo "===== DISK USAGE ====="

                    $ANSIBLE \
                    -i "$INVENTORY" \
                    rhel_servers \
                    -m command \
                    -a "df -h /"
                '''
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
            Required agents installed
            CloudWatch Agent configured and verified
            Both RHEL servers checked
            =========================================
            '''
        }

        failure {
            echo '''
            ANSIBLE PIPELINE FAILED.
            Please check the failed stage in Console Output.
            '''
        }

        always {
            echo 'Ansible pipeline execution finished.'
        }
    }
}

