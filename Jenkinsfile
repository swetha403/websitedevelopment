pipeline {
    agent any

    environment {
        // Define AWS and Chef paths or variables
        CHEF_HOME = '/home/jenkins/.chef'
        AWS_REGION = 'ap-south-1'
        EC2_USER = 'ubuntu'
        EC2_HOST = 'ec2-13-201-80-54.ap-south-1.compute.amazonaws.com'
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    // Clone the Git repository containing the website
                    checkout scm
                }
            }
        }

        stage('Package Application') {
            steps {
                powershell '''
                Write-Host "Packaging application..."
                Compress-Archive -Path * -DestinationPath website.zip -Force
                '''
            }
        }

        stage('Upload Package to EC2') {
            steps {
                sshagent(['aws-ec2-key']) {
                    powershell '''
                    Write-Host "Uploading package to EC2 instance..."
                    pscp -scp -pw $env:SSH_KEY_PATH website.zip $env:EC2_USER@$env:EC2_HOST:/tmp
                    '''
                }
            }
        }

        stage('Deploy with Chef') {
            steps {
                sshagent(['aws-ec2-key']) {
                    powershell '''
                    Write-Host "Setting up Chef and deploying application..."
                    ssh -o StrictHostKeyChecking=no $env:EC2_USER@$env:EC2_HOST "bash -s" <<'EOF'

                    # Install Chef if not present
                    if ! command -v chef-client &> /dev/null; then
                        echo "Installing Chef..."
                        curl -L https://omnitruck.chef.io/install.sh | sudo bash
                    fi

                    # Create the required directory structure
                    mkdir -p /tmp/cookbooks/website/recipes || exit 1
                    echo "Cookbook directory created: /tmp/cookbooks/website/recipes"

                    # Create the Chef configuration file for local mode
                    echo "
                    chef_license 'accept'
                    cookbook_path ['/tmp/cookbooks']
                    node_name 'website-deployment'
                    " | sudo tee /etc/chef/client.rb || exit 1

                    # Create the Chef recipe
                    echo "
                    bash 'unzip_website' do
                        code <<-EOH
                        # Ensure the target directory exists
                        sudo mkdir -p /var/www/html || exit 1

                        # Unzip the website.zip file to the target directory
                        sudo unzip -o /tmp/website.zip -d /var/www/html/ || exit 1

                        # Set the correct ownership for Apache (www-data on Ubuntu)
                        sudo chown -R www-data:www-data /var/www/html/
                        EOH
                    end
                    " | sudo tee /tmp/cookbooks/website/recipes/default.rb || exit 1

                    # Log to verify cookbook existence
                    ls -la /tmp/cookbooks/website/recipes
                    echo "Running Chef client in local mode..."

                    # Run Chef client in local mode
                    sudo chef-client --local-mode --runlist 'recipe[website]' -c /etc/chef/client.rb || exit 1
EOF'''
                }
            }
        }
    }

    post {
        success {
            Write-Host "Deployment completed successfully!"
        }
        failure {
            Write-Host "Deployment failed. Check logs for details."
        }
    }
}