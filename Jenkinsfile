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
                script {
                    echo 'Packaging application...'
                    // Use Groovy's zip method or call shell commands
                    sh 'zip -r website.zip *'
                }
            }
        }

        stage('Upload Package to EC2') {
            steps {
                sshagent(['aws-ec2-key']) {
                    sh '''
                    echo "Uploading package to EC2 instance..."
                    scp -o StrictHostKeyChecking=no website.zip $EC2_USER@$EC2_HOST:/tmp
                    '''
                }
            }
        }

        stage('Deploy with Chef')
        {
            steps {
                sshagent(['aws-ec2-key']) {
                    sh '''
                    echo "Setting up Chef and deploying application..."
                    ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "bash -s" <<'EOF'

                    # Ensure the target directory exists
                    sudo mkdir -p /var/www/html

                    # Set ownership and permissions
                    sudo chown -R www-data:www-data /var/www/html/
                    sudo chmod -R 755 /var/www/html/

                    # Install Apache and unzip
                    sudo apt-get update -y && sudo apt-get install -y apache2 unzip

                    # Ensure Apache is enabled and running
                    sudo systemctl enable apache2
                    sudo systemctl start apache2

                    # Install Chef if not already present
                    if ! command -v chef-client &> /dev/null; then
                        echo "Installing Chef..."
                        curl -L https://omnitruck.chef.io/install.sh | sudo bash
                    fi

                    # Create the required directory structure
                    sudo mkdir -p /tmp/cookbooks/website/recipes
                    echo "Cookbook directory created: /tmp/cookbooks/website/recipes"

                    # Create the Chef configuration file for local mode
                    echo "
                    chef_license 'accept'
                    cookbook_path ['/tmp/cookbooks']
                    node_name 'website-deployment'
                    " | sudo tee /etc/chef/client.rb

                    # Create the Chef recipe
                    echo "
                    bash 'unzip_website' do
                        code <<-EOH
                        
                        # Unzip the website.zip file to the target directory
                        sudo apt-get install -y unzip
                        sudo unzip -o /tmp/website.zip -d /var/www/html/

                        # Ensure the correct user exists for chown; use ubuntu as default
                        if id 'www-data' &>/dev/null; then
                            sudo chown -R www-data:www-data /var/www/html/
                        else
                            sudo chown -R ubuntu:ubuntu /var/www/html/
                        fi
                        EOH
                    end
                    " | sudo tee /tmp/cookbooks/website/recipes/default.rb

                    # Log to verify cookbook existence
                    ls -la /tmp/cookbooks/website/recipes
                    echo "Running Chef client in local mode..."

                    # Run Chef client in local mode
                    sudo chef-client --local-mode --runlist 'recipe[website]' -c /etc/chef/client.rb
EOF'''
                }
            }
        }
    }

    post {
        success {
            script {
                echo 'Deployment completed successfully!'
            }
        }
        failure {
            script {
                echo 'Deployment failed. Check logs for details.'
            }
        }
    }
}
