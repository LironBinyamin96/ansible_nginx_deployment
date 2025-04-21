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
        
        stage('Run Ansible Playbook') {
            steps {
                script {
                    // Execute the playbook directly on the remote server
                    sh """
                        # Copy the playbook to the remote server
                        scp -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem playbook-Nginx.yml ${VM_USER}@${VM_HOST}:~/playbook-Nginx.yml
                        
                        # Ensure Ansible is installed on the remote server
                        ssh -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem ${VM_USER}@${VM_HOST} '
                            if ! command -v ansible-playbook &> /dev/null; then
                                echo "Installing Ansible on the remote server..."
                                sudo apt-get update
                                sudo apt-get install -y ansible
                            fi
                            
                            # Create a simple localhost inventory file
                            echo "[webserver]" > ~/inventory
                            echo "localhost ansible_connection=local" >> ~/inventory
                            
                            # Run the playbook
                            ansible-playbook -i ~/inventory ~/playbook-Nginx.yml
                        '
                    """
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
        }
    }
}
