pipeline {
  agent any  // Execute the pipeline on any available Jenkins agent.

  stages {
    stage('Execute Ansible playbook') {  // Define the 'Execute Ansible playbook' stage.
      steps {
        sshagent(['controlNode']) {  // Start an SSH agent to securely connect to remote servers.

          // Clone a Git repository on the remote server using SSH.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} git clone https://github.com/GauravBarakoti/ansibleWithJenkins.git'

          // Execute an Ansible playbook on the remote server with environment variables.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} "cd ansibleWithJenkins; ansible-playbook playbook.yml -e region=$REGION -e instance_type=$INSTANCE -e image_id=$AMI -e key_name=$KEY -e subnet_id=$SUBNET -e aws_profile=$AWS_PROFILE;"'

          // Sleep for 40 seconds (this is for the Ansible playbook to complete ec2 povision tasks).
          sh 'sleep 40'

          // Execute another Ansible playbook on the remote server.
          sh 'ssh -tt -o StrictHostKeyChecking=no ubuntu@${ip} "cd ansibleWithJenkins;ansible-playbook webserver.yml;"'
        }
      }
    }
  }
}
