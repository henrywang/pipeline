pipeline {
    agent {
        // Stages run on the master by default.
        label 'master'
    }
    stages {
        stage('Install linchpin') {
            // This stage should be in its own job, which regularly updates the cinch install in the master.
            // Note that the ansible virtualenv is created just to ensure a supported version of ansible is
            // installed so that cinch itself, and its dependencies, can be installed. This virtualenv is
            // not needed by other jobs or stages, but the two virtualenvs that it creates *will* be needed
            // by other jobs and stages.
            steps{
                checkout scm
                sh """
                    virtualenv ansible.venv && source ansible.venv/bin/activate
                    pip install --upgrade setuptools pip
                    pip install 'ansible==2.4.1.0'

                    export PYTHONUNBUFFERED=1 # Enable real-time output for Ansible
                    ansible-playbook -i localhost, -c local test/end-to-end/openstack/playbooks/install-linchpin-rhel7.yml \
                        -e delete_venv="true"

                    deactivate
                """
            }
        }
        stage('Provision') {
            environment {
                LATEST_IMAGE = credentials('rhel_latest_image')
                OPENSTACK_AUTH_URL = credentials('openstack_auth_url')
                OPENSTACK_PROJECT_NAME = credentials('openstack_project_name')
                OPENSTACK_AUTH = credentials('openstack_auth')
                REPO_B = credentials('repo_b')
                REPO_O = credentials('repo_o')
                REPO_E = credentials('repo_e')
            }
            steps {
                dir('test/end-to-end/openstack') {
                    sh """
                        source "\${JENKINS_HOME}/opt/linchpin/bin/activate"
                        linchpin -v --creds-path credentials -w "\$(pwd)" up

                        chmod 600 keystore/ci-ops-central
                        ansible-playbook -v -i inventories/composer-test.inventory playbooks/provision.yml
                        deactivate
                    """
                }
            }
            post {
                // no post state for "not successful", but we want to skip the
                // test phase any time provisioning is not successful and mark
                // this build as having not been built in that case. The teardown
                // post-phase will always run regardless of the current result.
                always {
                    script {
                        if (currentBuild.currentResult != "SUCCESS") {
                            manager.buildNotBuilt()
                        }
                    }
                }
            }
        }
        stage('End to End Test') {
            steps {
                sh """
                    virtualenv ansible.venv && source ansible.venv/bin/activate
                    pip install --upgrade setuptools pip
                    pip install 'ansible==2.4.1.0'

                    export PYTHONUNBUFFERED=1 # Enable real-time output for Ansible
                    ansible-playbook -v -i test/end-to-end/openstack/inventories/composer-test.inventory openstack/playbooks/tests.yml \
                        -e delete_venv="true"

                    deactivate
                """
            }
            when {
                expression {
                    currentBuild.currentResult == "SUCCESS"
                }
            }
            post {
                // Always attempt to capture artifacts created during the test run, but
                // do not fail if they don't exist.
                always {
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'test/end-to-end/openstack/inventories/composer-test.inventory'
                }
            }
        }
    }
    post {
        // uses default 'master' agent, runs in same workspace and node as provision step
        always {
            // No matter what happens above, always attempt to destroy the deployed instance
            // entirely by running linchpin destroy.
            // dir('openstack') {
            //     sh """
            //         source "\${JENKINS_HOME}/opt/linchpin/bin/activate"
            //         linchpin -v --creds-path credentials -w "\$(pwd)" destroy || echo \
            //             "Linchpin destroy failed, VM may need to be manually removed from provider."
            //         deactivate
            //     """
            // }
            cleanWs()
        }
    }
    options {
        skipDefaultCheckout()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr:'10'))
        timeout(time: 6, unit: 'HOURS')
    }
}
