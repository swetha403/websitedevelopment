pipeline {
    agent any

    environment {
        CHEF_HOME = 'C:\\Users\\jenkins\\.chef'
        AWS_REGION = 'ap-south-1'
        EC2_USER = 'ubuntu'
        EC2_HOST = 'ec2-13-201-80-54.ap-south-1.compute.amazonaws.com'
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
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
                $ec2User = "${env:EC2_USER}"
                $ec2Host = "${env:EC2_HOST}"
                $destination = "${ec2User}@${ec2Host}:${remotePath}"

                # Use PSCP to transfer the file
                pscp -i $keyFile -scp website.zip $destination
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
                sudo chown -R www-data:www-data /var/www/html/
                sudo chmod -R 755 /var/www/html/
                sudo apt-get update -y && sudo apt-get install -y apache2 unzip
                sudo systemctl enable apache2
                sudo systemctl start apache2

                if ! command -v chef-client &> /dev/null; then
                    echo "Installing Chef..."
                    curl -L https://omnitruck.chef.io/install.sh | sudo bash
                fi

                sudo mkdir -p /tmp/cookbooks/website/recipes
                echo "
                chef_license 'accept'
                cookbook_path ['/tmp/cookbooks']
                node_name 'website-deployment'
                " | sudo tee /etc/chef/client.rb

                echo "
                bash 'unzip_website' do
                    code <<-EOH
                    sudo apt-get install -y unzip
                    sudo unzip -o /tmp/website.zip -d /var/www/html/
                    sudo chown -R www-data:www-data /var/www/html/
                    EOH
                end
                " | sudo tee /tmp/cookbooks/website/recipes/default.rb

                ls -la /tmp/cookbooks/website/recipes
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
