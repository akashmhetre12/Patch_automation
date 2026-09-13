pipeline {
    agent any

    parameters {
        // ---- The only things a person triggering this build should choose ----
        string(
            name: 'APP_GIT_BRANCH',
            defaultValue: 'main',
            description: 'Branch to checkout/pull from the application components repo'
        )
        string(
            name: 'PLAYBOOK_GIT_BRANCH',
            defaultValue: 'main',
            description: 'Branch to checkout/pull from the Ansible playbooks repo'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'qa', 'UAT', 'prod'],
            description: 'Target environment / host-group (--limit). Also selects inventory/<TARGET_ENV>.ini'
        )
        string(
            name: 'COMPONENTS',
            defaultValue: 'mcbatch,dbdelivery',
            description: 'Comma-separated component names to deploy. Must match folder names in the app components repo'
        )
    }

    environment {
        // ---- Fixed infrastructure config — not exposed as build parameters ----
        APP_GIT_REPO_URL      = 'https://github.com/akashmhetre12/Code_patch_deployment.git'
        PLAYBOOK_GIT_REPO_URL = 'https://github.com/akashmhetre12/Patch_automation.git'

        ANSIBLE_CONTROL_HOST  = '172.31.5.200'
        ANSIBLE_REMOTE_USER   = 'ubuntu'

        APP_REMOTE_DIR        = '/home/ubuntu/simple-java-project'
        PLAYBOOK_REMOTE_DIR   = '/home/ubuntu/Patch_automation'
        ANSIBLE_PLAYBOOK      = 'deploy.yml'

        // Jenkins credential ID for an SSH username+private key credential
        // that can log into the Ansible control node.
        SSH_CRED_ID = 'ansible-control-ssh-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: "*/main"]],
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
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_REMOTE_USER}@${ANSIBLE_CONTROL_HOST} '
                            set -e
                            if [ -d "${APP_REMOTE_DIR}/.git" ]; then
                                cd ${APP_REMOTE_DIR}
                                git fetch origin
                                git checkout ${params.APP_GIT_BRANCH}
                                git reset --hard origin/${params.APP_GIT_BRANCH}
                                git clean -fdx
                            else
                                rm -rf ${APP_REMOTE_DIR}
                                mkdir -p \$(dirname ${APP_REMOTE_DIR})
                                git clone -b ${params.APP_GIT_BRANCH} ${APP_GIT_REPO_URL} ${APP_REMOTE_DIR}
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
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_REMOTE_USER}@${ANSIBLE_CONTROL_HOST} '
                            set -e
                            if [ -d "${PLAYBOOK_REMOTE_DIR}/.git" ]; then
                                cd ${PLAYBOOK_REMOTE_DIR}
                                git fetch origin
                                git checkout ${params.PLAYBOOK_GIT_BRANCH}
                                git reset --hard origin/${params.PLAYBOOK_GIT_BRANCH}
                                git clean -fdx
                            else
                                rm -rf ${PLAYBOOK_REMOTE_DIR}
                                mkdir -p \$(dirname ${PLAYBOOK_REMOTE_DIR})
                                git clone -b ${params.PLAYBOOK_GIT_BRANCH} ${PLAYBOOK_GIT_REPO_URL} ${PLAYBOOK_REMOTE_DIR}
                            fi
                        '
                    """
                }
            }
        }

        stage('Verify Inventory Resolves Hosts') {
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_REMOTE_USER}@${ANSIBLE_CONTROL_HOST} '
                            set -e
                            cd ${PLAYBOOK_REMOTE_DIR}
                            echo "Checking inventory file exists:"
                            ls -la inventory/${params.TARGET_ENV}.ini
                            echo "Hosts matched for --limit ${params.TARGET_ENV}:"
                            MATCHED=\$(ansible-inventory -i inventory/${params.TARGET_ENV}.ini --list --limit ${params.TARGET_ENV} | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get(\\"_meta\\",{}).get(\\"hostvars\\",{})))")
                            echo "Matched host count: \$MATCHED"
                            if [ "\$MATCHED" -eq 0 ]; then
                                echo "ERROR: No hosts matched inventory/${params.TARGET_ENV}.ini limit=${params.TARGET_ENV}"
                                exit 1
                            fi
                        '
                    """
                }
            }
        }

        stage('Approval') {
            steps {
                script {
                    timeout(time: 30, unit: 'MINUTES') {
                        input(
                            id: 'DeployApproval',
                            message: "Approve deployment of components [${params.COMPONENTS}] to ${params.TARGET_ENV}? (app branch: ${params.APP_GIT_BRANCH}, playbook branch: ${params.PLAYBOOK_GIT_BRANCH})",
                            ok: 'Deploy'
                            // Optional: restrict who can approve, e.g.:
                            // submitter: 'admin,deploy-team'
                        )
                    }
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_REMOTE_USER}@${ANSIBLE_CONTROL_HOST} \\
                            "cd ${PLAYBOOK_REMOTE_DIR} && \\
                             ansible-playbook -i inventory/${params.TARGET_ENV}.ini ${ANSIBLE_PLAYBOOK} \\
                             --limit ${params.TARGET_ENV} \\
                             --extra-vars 'deploy_components=${params.COMPONENTS} artifact_dir=${APP_REMOTE_DIR}'"
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
}
