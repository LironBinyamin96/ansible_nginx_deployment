pipeline {
    agent any
    environment {
        VAULT_TOKEN = credentials('vault-token-secret-text')
        VM_HOST = 'Liron.aws.cts.care'
        VM_USER = 'ubuntu'
    }
    stages {
        stage('Get SSH Private Key from Vault') {
            steps {
                script {
                    // Create a directory for storing credentials securely
                    sh 'mkdir -p ~/.ssh_temp && chmod 700 ~/.ssh_temp'
                    
                    // Retrieve private key from Vault using direct curl with basic shell processing
                    withCredentials([string(credentialsId: 'vault-token-secret-text', variable: 'VAULT_TOKEN')]) {
                        sh '''
                            # Get response from Vault 
                            RESPONSE=$(curl --silent --header "X-Vault-Token: $VAULT_TOKEN" --request GET http://vault:8200/v1/secret/aws/privat-key)
                            
                            # Extract the private key using basic shell commands
                            echo "$RESPONSE" | sed 's/.*"value":"//' | sed 's/".*//' > ~/.ssh_temp/ssh_key.pem
                            
                            # Fix newlines (replace \\n with actual newlines)
                            sed -i 's/\\\\n/\\n/g' ~/.ssh_temp/ssh_key.pem
                            
                            # Set correct permissions
                            chmod 600 ~/.ssh_temp/ssh_key.pem
                        '''
                    }
                    
                    echo "Private Key retrieved and stored securely"
                    
                    // Test connection to VM
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem ${VM_USER}@${VM_HOST} 'echo "Successfully connected to the virtual machine!"'
                    """
                }
            }
        }
        
        stage('Install Ansible') {
            steps {
                script {
                    // Ensure Ansible and ansible-galaxy are installed on the Jenkins host
                    sh '''
                        if ! command -v ansible-playbook &> /dev/null; then
                            echo "Installing Ansible on Jenkins host..."
                            apt-get update || true
                            apt-get install -y ansible || true
                        fi
                        
                        # Check if ansible-galaxy is available
                        if ! command -v ansible-galaxy &> /dev/null; then
                            echo "ansible-galaxy not found. Ansible installation may be incomplete."
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Create Ansible Inventory') {
            steps {
                script {
                    // Create inventory file for Ansible to target the remote server
                    sh '''
                        mkdir -p ansible
                        echo "[webserver]" > ansible/inventory
                        echo "${VM_HOST} ansible_user=${VM_USER} ansible_ssh_private_key_file=~/.ssh_temp/ssh_key.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> ansible/inventory
                        cat ansible/inventory
                    '''
                }
            }
        }
        
        stage('Run Ansible Playbook') {
            steps {
                script {
                    // Run ansible-galaxy to install required roles
                    sh '''
                        cd ansible
                        # Install required roles from requirements file if it exists
                        if [ -f "requirements.yml" ]; then
                            ansible-galaxy install -r requirements.yml
                        fi
                    '''
                    
                    // Execute the playbook from the Jenkins host targeting the remote VM
                    sh '''
                        ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i ansible/inventory nginx.yml -v
                    '''
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                // Clean up the SSH key after use
                sh 'rm -rf ~/.ssh_temp'
                echo "Credentials cleaned up"
            }
        }
    }
    
    post {
        always {
            // Ensure credentials are always cleaned up, even if the pipeline fails
            sh 'rm -rf ~/.ssh_temp || true'
            sh 'rm -rf ansible || true'
        }
    }
}
