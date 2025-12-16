pipeline {
    agent {
        docker { 
            image 'willhallonline/ansible:latest'
            args '--network host -u root' 
        }
    }

    environment {
        SSH_KEY = credentials('ansible-ssh-key')
        VAULT_PASS = credentials('ansible-vault-pass')
        ANSIBLE_FORCE_COLOR = 'true'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        INVENTORY = 'inventory.ini'
    }

    stages {
        stage('Init') {
            steps {
                sh 'ansible-galaxy collection install ansible.posix'
                sh 'chmod -R 755 .'
                sh 'echo $VAULT_PASS > .vault_pass'
                sh 'chmod 600 .vault_pass'
            }
        }

        stage('Lint: All') {
            steps {
                withEnv(['ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass']) {
                    sh "ansible-playbook -i ${INVENTORY} setup_env.yml --syntax-check"
                    sh "ansible-playbook -i ${INVENTORY} site.yml --syntax-check"
                    script {
                        try {
                            sh 'ansible-lint setup_env.yml site.yml'
                        } catch (Exception e) {
                            echo "Linting issues found."
                        }
                    }
                }
            }
        }

        stage('Plan: Setup') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh 'mkdir -p ~/.ssh'
                    sh 'ssh-keygen -y -f "$SSH_KEY_FILE" > ~/.ssh/id_rsa.pub'
                    sh "ansible-playbook -i ${INVENTORY} setup_env.yml --private-key \"$SSH_KEY_FILE\" --check"
                }
            }
        }

        stage('Deploy: Setup') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh "ansible-playbook -i ${INVENTORY} setup_env.yml --private-key \"$SSH_KEY_FILE\""
                }
            }
        }

        stage('Plan: Users & App') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh "ansible-playbook -i ${INVENTORY} site.yml --private-key \"$SSH_KEY_FILE\" --check"
                }
            }
        }

        stage('Deploy: Users & App') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh "ansible-playbook -i ${INVENTORY} site.yml --private-key \"$SSH_KEY_FILE\""
                }
            }
        }

        stage('Verify') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh "ansible -i ${INVENTORY} webservers -m uri -a 'url=http://localhost:8080 return_content=yes status_code=200' --private-key \"$SSH_KEY_FILE\""
                }
            }
        }
    }
}