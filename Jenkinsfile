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
        PEM_KEY = '/var/lib/jenkins/workspace/Terraform/rhel-terraform-key.pem'
    }

    stages {

        stage('Checkout Ansible Repository') {
            steps {
                checkout scm
                echo 'Ansible repository checkout completed.'
            }
        }

        stage('Check Tools and Files') {
            steps {
                sh '''
                    set -e

                    echo "Checking Ansible version..."
                    ${ANSIBLE} --version

                    echo "Checking required files..."

                    test -f ${INVENTORY}
                    test -f users.yml
                    test -f filesystem.yml
                    test -f swap.yml
                    test -f agents.yml
                    test -f cloudwatch-agent.yml

                    echo "Checking SSH private key..."

                    test -r ${PEM_KEY}

                    echo "All required tools and files are available."
                '''
            }
        }

        stage('Ansible Connectivity Test') {
            steps {
                sh '''
                    set -e

                    echo "Testing connectivity to RHEL servers..."

                    ${ANSIBLE} -i ${INVENTORY} rhel_servers -m ping
                '''
            }
        }

        stage('Create Linux Users') {
            steps {
                sh '''
                    set -e

                    echo "Creating Linux users..."

                    ${ANSIBLE_PLAYBOOK} -i ${INVENTORY} users.yml
                '''
            }
        }

        stage('Configure Filesystems and Folders') {
            steps {
                sh '''
                    set -e

                    echo "Configuring filesystems and folders..."

                    ${ANSIBLE_PLAYBOOK} -i ${INVENTORY} filesystem.yml
                '''
            }
        }

        stage('Configure Swap') {
            steps {
                sh '''
                    set -e

                    echo "Configuring swap..."

                    ${ANSIBLE_PLAYBOOK} -i ${INVENTORY} swap.yml
                '''
            }
        }

        stage('Install Required Agents') {
            steps {
                sh '''
                    set -e

                    echo "Installing required agents..."

                    ${ANSIBLE_PLAYBOOK} -i ${INVENTORY} agents.yml
                '''
            }
        }

        stage('Install and Configure CloudWatch Agent') {
            steps {
                sh '''
                    set -e

                    echo "Installing and configuring CloudWatch Agent..."

                    ${ANSIBLE_PLAYBOOK} -i ${INVENTORY} cloudwatch-agent.yml
                '''
            }
        }

        stage('Verify CloudWatch Agent') {
            steps {
                sh '''
                    set -e

                    echo "Checking CloudWatch Agent service..."

                    ${ANSIBLE} -i ${INVENTORY} rhel_servers \
                        -m shell \
                        -a "systemctl is-active amazon-cloudwatch-agent"
                '''
            }
        }

        stage('Final Server Verification') {
            steps {
                sh '''
                    set -e

                    echo "Final server verification..."

                    ${ANSIBLE} -i ${INVENTORY} rhel_servers \
                        -m shell \
                        -a "hostname; uptime; df -h"

                    echo "Server verification completed."
                '''
            }
        }
    }

    post {
        success {
            echo '''
            ==========================================
            ANSIBLE AUTOMATION COMPLETED SUCCESSFULLY
            ==========================================
            Linux users configured
            Filesystems configured
            Swap configured
            Required agents installed
            CloudWatch Agent verified
            Server verification completed
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
            ANSIBLE AUTOMATION FAILED
            Please check the failed stage and console output.
            ==========================================
            '''
        }

        always {
            echo 'Ansible pipeline execution finished.'
        }
    }
}
