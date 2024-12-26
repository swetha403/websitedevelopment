pipeline {
    agent any

    environment {
        // Define AWS and EC2 details
        CHEF_HOME = 'C:\\Users\\jenkins\\.chef'
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
                Write-Host "Cleaning up existing ZIP file..."
                if (Test-Path -Path "website.zip") {
                    Remove-Item -Path "website.zip" -Force
                }

                Write-Host "Packaging application..."
                Compress-Archive -Path * -DestinationPath website.zip
                '''
            }
        }

        stage('Upload Package to EC2') {
    steps {
        powershell '''
        Write-Host "Uploading package to EC2 instance using PSCP..."
        $keyFile = "C:\\keys\\aws-ec2-key.ppk"
        $remotePath = "/tmp/website.zip"
        $ec2User = "$env:EC2_USER@$env:EC2_HOST"

        # Use PSCP to transfer the file
        pscp -i $keyFile -scp website.zip $ec2User:$remotePath
        '''
    }
}


        stage('Deploy with Chef') {
            steps {
                powershell '''
                Write-Host "Deploying application on EC2 with Chef..."
                plink -i aws-ec2-key.ppk -hostkey aa:bb:cc:dd:ee:ff:gg:hh $env:EC2_USER@$env:EC2_HOST "bash -s" <<'EOF'
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
                EOF
                '''
            }
        }
    }

    post {
        success {
            powershell '''
            Write-Host "Deployment completed successfully!"
            '''
        }
        failure {
            powershell '''
            Write-Host "Deployment failed. Check logs for details."
            '''
        }
    }
}
