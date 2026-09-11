pipeline {
    agent any

    parameters {
        // ---- Application / components repo ----
        string(
            name: 'APP_GIT_REPO_URL',
            defaultValue: 'https://github.com/akashmhetre12/simple-java-project.git',
            description: 'Git repo holding the unzipped application components (mcbatch, dbdelivery, etc.)'
        )
        string(
            name: 'APP_GIT_BRANCH',
            defaultValue: 'main',
            description: 'Branch to checkout/pull from the application components repo'
        )

        // ---- Ansible playbooks / inventory repo ----
        string(
            name: 'PLAYBOOK_GIT_REPO_URL',
            defaultValue: 'https://github.com/akashmhetre12/Patch_automation.git',
            description: 'Git repo holding the Ansible playbooks and inventory files'
        )
        string(
            name: 'PLAYBOOK_GIT_BRANCH',
            defaultValue: 'main',
            description: 'Branch to checkout/pull from the Ansible playbooks repo'
        )

        // ---- Ansible control node ----
        string(
            name: 'ANSIBLE_CONTROL_HOST',
            defaultValue: '172.31.5.200',
            description: 'Hostname or IP of the Ansible control node'
        )
        string(
            name: 'ANSIBLE_REMOTE_USER',
            defaultValue: 'ubuntu',
            description: 'SSH user on the Ansible control node'
        )
        string(
            name: 'APP_REMOTE_DIR',
            defaultValue: '/home/ubuntu/simple-java-project',
            description: 'Directory on the control node where the APP repo is checked out (this becomes artifact_dir passed to Ansible)'
        )
        string(
            name: 'PLAYBOOK_REMOTE_DIR',
            defaultValue: '/home/ubuntu/Patch_automation',
            description: 'Directory on the control node where the PLAYBOOK repo is checked out'
        )
       
        string(
            name: 'ANSIBLE_PLAYBOOK',
            defaultValue: 'deploy.yml',
            description: 'Path RELATIVE to PLAYBOOK_REMOTE_DIR to the deployment playbook'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'qa', 'UAT', 'prod'],
            description: 'Target environment / host-group in the inventory (--limit)'
        )
        string(
            name: 'COMPONENTS',
            defaultValue: 'mcbatch,dbdelivery',
            description: 'Comma-separated component names to deploy. Must match folder names in the app components repo'
        )
    }

    environment {
        // Jenkins credential ID for an SSH username+private key credential
        // that can log into the Ansible control node.
        SSH_CRED_ID = 'ansible-control-ssh-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: "*/${params.BRANCH}"]],
                    userRemoteConfigs: [[url: 'https://github.com/akashmhetre12/Patch_automation.git']]
                )
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!params.COMPONENTS?.trim()) {
                        error "COMPONENTS must not be empty. Example: mcbatch,dbdelivery"
                    }
                    if (!params.APP_GIT_BRANCH?.trim() || !params.PLAYBOOK_GIT_BRANCH?.trim()) {
                        error "APP_GIT_BRANCH and PLAYBOOK_GIT_BRANCH must not be empty."
                    }
                    echo "App repo branch: ${params.APP_GIT_BRANCH}"
                    echo "Playbook repo branch: ${params.PLAYBOOK_GIT_BRANCH}"
                    echo "Target environment: ${params.TARGET_ENV}"
                    echo "Components requested: ${params.COMPONENTS}"
                }
            }
        }

        stage('Checkout Application Components on Control Node') {
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${params.ANSIBLE_REMOTE_USER}@${params.ANSIBLE_CONTROL_HOST} '
                            set -e
                            if [ -d "${params.APP_REMOTE_DIR}/.git" ]; then
                                cd ${params.APP_REMOTE_DIR}
                                git fetch origin
                                git checkout ${params.APP_GIT_BRANCH}
                                git reset --hard origin/${params.APP_GIT_BRANCH}
                                git clean -fdx
                            else
                                rm -rf ${params.APP_REMOTE_DIR}
                                mkdir -p \$(dirname ${params.APP_REMOTE_DIR})
                                git clone -b ${params.APP_GIT_BRANCH} ${params.APP_GIT_REPO_URL} ${params.APP_REMOTE_DIR}
                            fi
                        '
                    """
                }
            }
        }

        stage('Checkout Ansible Playbooks on Control Node') {
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${params.ANSIBLE_REMOTE_USER}@${params.ANSIBLE_CONTROL_HOST} '
                            set -e
                            if [ -d "${params.PLAYBOOK_REMOTE_DIR}/.git" ]; then
                                cd ${params.PLAYBOOK_REMOTE_DIR}
                                git fetch origin
                                git checkout ${params.PLAYBOOK_GIT_BRANCH}
                                git reset --hard origin/${params.PLAYBOOK_GIT_BRANCH}
                                git clean -fdx
                            else
                                rm -rf ${params.PLAYBOOK_REMOTE_DIR}
                                mkdir -p \$(dirname ${params.PLAYBOOK_REMOTE_DIR})
                                git clone -b ${params.PLAYBOOK_GIT_BRANCH} ${params.PLAYBOOK_GIT_REPO_URL} ${params.PLAYBOOK_REMOTE_DIR}
                            fi
                        '
                    """
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${params.ANSIBLE_REMOTE_USER}@${params.ANSIBLE_CONTROL_HOST} \\
                            "cd ${params.PLAYBOOK_REMOTE_DIR} && \\
                             ansible-playbook -i inventory/${params.TARGET_ENV} ${params.ANSIBLE_PLAYBOOK} \\
                             --limit ${params.TARGET_ENV} \\
                             --extra-vars 'deploy_components=${params.COMPONENTS} artifact_dir=${params.APP_REMOTE_DIR}'"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment of [${params.COMPONENTS}] (app branch '${params.APP_GIT_BRANCH}', playbook branch '${params.PLAYBOOK_GIT_BRANCH}') to ${params.TARGET_ENV} completed successfully."
        }
        failure {
            echo "Deployment failed — check the console log above for the failing stage."
        }
        always {
            cleanWs()
        }
    }
}
