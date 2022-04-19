# k8s-configuration
Personal K8s Config Folder
DEPLOYING K8S WITH TERRAFORM END TO END WITH RELATIVE EASE

- Create a Ubuntu 18.04 server (Can be t2.micro or t2.medium and give it a name of choice) open necessary ports (all ports for practice but SSH in prod environment)

- SSH into your Ubuntu server with the necessary credentials (preferably with VSCode GitBash)

- Update Ubuntu server with: 
   $ sudo apt-get update
  
- Create a new user with: 
   $ sudo adduser eksadmin 
  You have to create a password and then just press enter for all other parameters.

- Add eksadmin user to sudoers group with the command:
   $ sudo echo "eksadmin  ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/eksadmin

- Switch to eksadmin user: 
   $ sudo su - eksadmin

- On your AWS console, navigate to IAM (Services menu bar -> IAM) to create a new role for AWS service - You can name it eksiam. This role will create access for Terraform to create your cloud infrastructure.
    IAM -> Roles -> Create role -> AWS service. Click EC2 as a common use case and click next. 
    In the search bar for policies: 
      Search for AmazonEC2FullAccess and check
      Search for AmazonVPCFullAccess and check also
      Search for AmazonS3FullAccess and check
      Search for AdministratorAccess and check #the administrator access policy alone covers for the other 3 but it's good to familiarize with the AWS IAM module. In an actual prod environment, it's a security vulnerability to grant admin access, you'll need to select the policies you need which I believe is the top 3.
    # With this, you will have have granted your role full access to 4 AWS services
    Click on next then enter your preferred role name (in our case - eksiam) and complete by clicking create role.

- Navigate to your Ubuntu instance on AWS console (Services -> EC2 -> Instances -> *your Ubuntu instance*)
    Click on Actions -> Security -> Modify IAM role and choose eksiam in the dropdown for IAM roles then save.

# Return to your VSCode terminal to configure AWS CLI

- Install AWS CLI with: 
   $ sudo apt-get install awscli
  
- Configure your AWS CLI with the command:
   $ aws configure

    Just press enter (leave it blank) when it asks for Access Key ID and Secret Access Key (These will be taken from the role we already attached to our Ubuntu server)
    Enter a preferred default region name but better it's in a different region so you can see what is created uniquely by Terraform. If your Ubuntu server is on Ohio (us-east-2) enter us-west-2 (Oregon) or any other region of choice for your default region name.
    For default output format, you can enter JSON or leave it blank

- Clone your terraform scripts from GIT into a folder in your Ubuntu server 
    $ git clone https://github.com/mylandmarktechs/eks-terraform-setup

  This will create a folder - eks-terraform-setup in your home folder (you can perform command ls to confirm). Rename it to shorten characters :)
   $ mv eks-terraform-setup ek

- Navigate into your ek folder, it contains all your Terraform scripts

    $ cd ek

- Install Terraform on your Ubuntu server with the following command
# Note that this is the old version of Terraform and it will flag it but it's the compatible version with Prof's terraform scripts

  $ sh terraform-install.sh

# (Quick detour for a requirement to make your Terraform scripts work)
-  Navigate to the AWS region you have decided to use as your EKS nodes host and create a key pair, in our case - us-west-2 (Oregon)
      Region -> Oregon -> EC2 Dashboard -> Key pairs -> Create key pair
      Give your key pair a name - ekskey (can be any name but has to be equal to what is in variables.tf). 
      Chose RSA type and .pem as private key file format then click on create key pair.

- Back to your ek folder on the Ubuntu terminal, vi into the file variables.tf and ensure that your key pair name that you created is the same as what is in the key_pair_name variable (in our case ekskey).
    $ vi variables.tf

- Also vi into providers.tf and ensure that the region you have chosen is the same as what is in the file (in our case us-west-2).
    $ vi providers.tf
  
# After ensuring all the config files are ready
- Run the command to initialize terraform
   $ terraform init 

- Then run the command to ensure that the configuration in the ek folder for scripts is valid.
   $ terraform validate

- Run the command for terraform to display a layout of the infrastructure to be created
   $ terraform plan
#  Important to note that terraform has an intelligent system to arrange all tf files in the ek folder and arrange logically without you specifying e.g. VPC has to be created before the EC2 instances can be created hence no need for an index file or main.tf file

- Create your AWS infrastructure on Terraform with the command 
   $ terraform apply
  This command will first plan your infrastructure and ask you to enter "yes" to approve the infrastructure creation.
# The terraform apply (infrastructure creation command) takes time to provision the entire infrastructure. IT MAY TAKE 11-15 MINUTES. Don't worry about it if you configured well, just grab popcorn and watch.
#
# terraform state list

# terraform state show aws_vpc.demo
# When you run the script to create EKS with terraform, by default it comes with one VPC, 2 private subnets, 2 public subnets, security groups (firewalls) and a managed control plane with 3 master nodes in the 3 AZs of your region for HIGH AVAILABILITY.

- Install kubectl in order to remotely connect to your Terraform provisioned EKS infrastructure from your Ubuntu server with the following commands. 
    $ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

    $ chmod +x ./kubectl

    $ sudo mv ./kubectl /usr/local/bin/kubectl

- Connect to your EKS cluster from your Ubuntu server by running the command (after your EKS cluster has been created as you need the cluster name - it's already in the variables.tf file though)
    $ aws eks --region us-west-2 update-kubeconfig --name terraform-eks-demo

  You will have a confirmation with arn number to show that you are connected now

- Test your connection to the EKS server by running the command 
    $ kubectl get all

# ================================================================================
# End of Terraform Infrastructure Provisioning, Next is to deploy Landmark Springboot App
#================================================================================

- Use wget to download the Yaml file for Landmark's Springboot app into our ek directory with below command 
    $  wget https://raw.githubusercontent.com/aanuoluwapoakinyera/kubernetes-manifests/main/SpringBoot-Mongo-DynamicPV.yml

- Rename the Yaml file to something shorter to avoid typo errors
    $  mv SpringBoot-Mongo-DynamicPV.yml sbapp.yaml

- Deploy your Landmark Springboot app on your K8s cluster 
    $ kubectl apply -f sbapp.yaml

- Get the external IP of the Springboot application in order to be able to assess it on a browser by executing the command: 
    kubectl get service springapp
   Copy the url from the External IP column and paste in a browser. It should come in this format: aaaaaabbbbbbcccccc-dddddd.us-west-1.elb.amazonaws.com
   # You can also find it in Amazons Elastic Load Balancer section by navigating to EC2 -> Load balancers (on the left menu)
   Once you copy this URL, you will be able to assess the springboot Landmark application and VOILA, congratulations. 

# REMEMBER TO DESTROY YOUR INFRASTRUCTURE WITH terraform destroy TO AVOID EXCESSIVE BILLING FROM AWS. YOU CAN ALWAYS CREATE A NEW INFRASTRUCTURE BY RUNNING YOUR SCRIPT.
