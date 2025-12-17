pipeline {
    agent any

    environment {
        VAULT_ADDR = 'http://192.168.1.26:8200'
        REMOTE_HOST = '192.168.1.24'
        REMOTE_USER = 'kokou'
        REMOTE_APP_DIR = '/opt/yara_app'
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/kokoudonne/jenkins.git'
            }
        }

        stage('Load secrets from Vault') {
            steps {
               jenkinsfile from vault
                withVault([
                    vaultSecrets: [[
                        path: 'secret/yara_app',
                        secretValues: [
                            [envVar: 'WORDPRESS_DB_HOST', vaultKey: 'db_host'],
                            [envVar: 'WORDPRESS_DB_USER', vaultKey: 'db_user'],
                            [envVar: 'WORDPRESS_DB_PASSWORD', vaultKey: 'db_password'],
                            [envVar: 'WORDPRESS_DB_NAME', vaultKey: 'db_name']
                        ]
                    ]]
                ]) {
                    sh 'echo "Secrets successfully loaded from Vault"'
                }
            }
        }

        stage('Sync docker-compose to remote server') {
            steps {
                sshagent(credentials: ['ssh-yara_app']) {
                    sh """
                    ssh ${REMOTE_USER}@${REMOTE_HOST} 'mkdir -p ${REMOTE_APP_DIR}'
                    rsync -avz docker-compose.yml ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_APP_DIR}/
                    """
                }
            }
        }

        stage('Deploy WordPress on remote server') {
            steps {
                sshagent(credentials: ['ssh-yara_app']) {
                    sh """
                    ssh ${REMOTE_USER}@${REMOTE_HOST} << 'EOF'
                    export WORDPRESS_DB_HOST='${WORDPRESS_DB_HOST}'
                    export WORDPRESS_DB_USER='${WORDPRESS_DB_USER}'
                    export WORDPRESS_DB_PASSWORD='${WORDPRESS_DB_PASSWORD}'
                    export WORDPRESS_DB_NAME='${WORDPRESS_DB_NAME}'

                    cd ${REMOTE_APP_DIR}
                    docker compose down || true
                    docker compose up -d
                    EOF
                    """
                }
            }
        }

        stage('Verify deployment') {
            steps {
                sshagent(credentials: ['ssh-yara_app']) {
                    sh "ssh ${REMOTE_USER}@${REMOTE_HOST} docker ps | grep yarashop"
                }
            }
        }
    }

    post {
        success {
            echo '✅ WordPress déployé avec succès sur 192.168.1.24 (Vault OK)'
        }
        failure {
            echo '❌ Échec du déploiement WordPress'
        }
    }
}
