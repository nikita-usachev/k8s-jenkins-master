# Jenkins Master on Kubernetes 

Deploy a configuration-automated Jenkins Master on an EKS cluster with AWS Parameter Store secrets injected into newly created pods.

## Services available:

* Kubernetes manifests that are used to deploy the Jenkins Master (including Service, StatefulSet, ClusterRoleBinding, Ingress, etc...)
* Makefile to simply create and manage Jenkins with targets.
* Dockerfile, which creates a custom image with Jenkins Configuration as Code (JCaC).
* Dockerfile.ja, which creates a basic image for our agent pods.
* jenkins-config.yaml, which contains the necessary code to preconfigure Jenkins with usernames and passwords for three users, along with the necessary plugins. This file will also contain JCaC code that allows our Jenkins Master to spin up new agents as needed.
* required plugins inserted into plugins.txt, which is installed via Dockerfile.

## Prerequisites:

* Install aws cli:
```
pip install awscli
```
* Install kubectl:
```
brew install kubernetes-cli
```
* Install helm 
```
brew install helm
```

* Install Secret Store Driver and ASCP 
	* CSI Driver is a third party tool that allows us to retrieve secrets from AWS Secrets Manager or Parameter Store and sync them as Kubernetes Secrets. ASCP is the AWS Provider for CSI Driver and it allows us to mount our Parameter Store secrets inside of our pod via a mounted file.
	* You will need to add ```--set syncSecret.enabled=true``` to your csi driver installation. This allows for your secret specified in SecretProviderClass to be automatically created.
	* NOTE: There is also a secondary way to set up CSI Driver and ASCP through yaml files, but for this I used the helm installations.
```
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts 
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws 
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```

## What is Jenkins Configuration as Code (JCaC)?

Jenkins Configuration as Code is a Jenkins plugin that allows for the setup of Jenkins to be pre-configured. This can save time and also create a reusable template for anyone that might want to test some other features out. It allows you to bootstrap a whole new Jenkins setup with any configuration that they want. In our case, we used it to preconfigure three users with passwords and their respective permissions. We also used it to configure the basic specifications for the agent pod that gets spun up for a new job.

## Steps to Create/How to Use:

1. Create a Jenkins configuration file that automates the creation of users and installs plugins.
	* plugins.txt file should be created with all of the necessary plugins for JCaC.
		* The key plugins for us will be configuration as code, kubernetes, and github.
	* jenkins-config.yaml should be created with code that creates users, passwords and their respective permissions. This file should also contain the proper code that allows our Jenkins Master to spin up new agents as needed.

2. Create a Dockerfile with the base as a current jenkins version.
	* Disable the setup wizard in your Docker image, as well. This will get rid of the preconfiguration screen that requires you to provide where you want to install Jenkins on your machine, as well as our initial login information. 
		* ```ENV  JAVA_OPTS  -Djenkins.install.runSetupWizard=false``` inside of Dockerfile
	* When then set our ENV to ```CASC_JENKINS_CONFIG```, because this is ENV that is used to look for plugins that are being installed. 
	*  Make sure to ```COPY``` this jenkins-config.yaml to /var/jenkins_home directory. This directory is where Jenkins configurations are applied.
	* ```Copy``` plugins.txt to /usr/share/jenkins/ref/
	* install-plugins.sh is deprecated so you will need to use jenkins-plugin-cli for the installation of plugins.

3. You'll need to create a second Docker image specifically for your agent that will be spun up when a new job is created. 
	* For this, we will use a small base OS (alpine) and we will install some basic tools that our agent will need.
	* Make sure to include the installation of basic commands that might be used (AWS, python, bash, etc...)
	* For this step, I created Dockerfile.ja, and specified the filename when building the image.

4. Build docker image using the supplied Dockerfile
```
docker build -t REPO/IMAGE_NAME:TAG .
```
* NOTE: If you use a MacBook with an M1 chip, you might need to adjust the architecture that your image is built on. M1 chip by default creates images using arm64 architecture. Since we are using linux ec2 instances, we will need to adjust this.
```
docker build --platform linux/amd64 -t repo/image_name .
```
* The above command will allow for your image to be built on the proper architecture.
* For agent image, this is an example of the command you can run to build off of the new Dockerfile.ja:
	*	```docker build --platform linux/amd64 -t repo/image_name:tag -f Dockerfile.ja .```
		*	```-f``` in this command specifies the Dockerfile that you want to create your new build with.

5. Test your docker image by running a docker container with the previously created image
```
docker run --name jenkins --rm -p 8080:8080 image_name
```
* In this command, ``` --rm ``` instructs the container to be deleted immediately upon stoppage of the container.
* Log into Jenkins UI with localhost and confirm that plugins and users were preconfigured.
* Your image is all set for the next steps! Push it to DockerHub!

6. Create your secrets for usernames and passwords for each user inside of AWS Parameter Store.
* For this step, you can use create a new parameter for each variable needed along with SecureString.
	* ex: ```/dev/jenkins/ADMIN_PASS```

7. You will need to create an OpenIDConnect (OIDC) that connects to your cluster. This step allows for our third party tool CSI Driver to be authenticated and access AWS Parameter Store parameters for use inside of our cluster.
* For our OIDC, when prompted to input your audience, enter ```sts.amazonaws.com```
* Steps to achieve this will be:
	* Go to EKS -> Click on the cluster that you are using -> Copy the Open ID Connect Provider URL
	* Go to IAM ->  Identity Providers (click Add Provider) -> Select OpenIDConnect -> Paste your EKS OIDC ARN that you copied into Provider URL -> Audience should be sts.amazonaws.com.

8. Next, you will need to create an IAM Policy with "Action: ssm:GetParameters", "Action:ssm:GetParametersByPath", "Action:ssm:GetParameter" with the attached resource being the arn of your newly created parameters.
* This would look something like this:
``` 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParametersByPath",
                "ssm:GetParameters",
                "ssm:GetParameter"
            ],
            "Resource": [
                "arn:aws:ssm:region_here:account_number:parameter/path_to_parameter"
            ]
        }
    ]
}
```

9. Create a Role and attach the previously created Policy to the Role. Modify the trust relationships to include your OIDC arn under the "Federated" category. Along with this, adjust your "StringEquals" line and change "aud" to "sub". After this, adjust "sts.amazonaws" to "system:serviceaccount:namespace_name:service_account_name. This step allows your service account to assume your created Role.
* You can run this command to find your OIDC ARN:
	* ```aws iam list-open-id-connect-providers```
*   Your trust policy will end up looking something like this:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "OIDC_ARN_HERE"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-1.amazonaws.com/id/9EAF82963CA771E4EC69F351AFE2C1DB:sub": "system:serviceaccount:namespace_here:service_account_name",
                }
            }
        }
    ]
}
```
* In this trust relationship, our OIDC provider is allowed to assume only the role of our service account.
* For service account slot, it will just be a reference to the name of your own service account.

10. Next, we need to create our Kubernetes objects. For this, you will need to create a ClusterRole, ClusterRoleBinding, Ingress, Service, StatefulSet, and a SecretProviderClass. Refer to k8s objects inside of this repo for code.
	* ClusterRole will include the necessary permissions to retrieve the secret values and create agent pods as needed.
	* ClusterRoleBinding allows permissions to be granted cluster-wide.
	* Ingress to allow our own DNS record to be used and exposed for our Jenkins Master from the outside.
	* Service to expose our pods inside the cluster.
	* StatefulSet for the deployment of our actual Jenkins Master. This StatefulSet must also contain a volume and volumeMount, which will mount the secret on our pod as a file. We will also include ENVs which will allow us to have our secret values inside.
		* NOTE: For your ENV variables inside of your pod, make sure your secretKeyRef matches the name of your synced secret master-secret. Your key should match the objectAlia that you specified inside of your secretProviderClass parameters.
	* SecretProviderClass, which specifies which secret(s) will be mounted inside of our pods. This will also sync to create a Kubernetes secret. Some tips:
		* Inside of parameters section in SecretProviderClass, make sure objectName is the path to your parameter inside of AWS Parameter Store. Your objectAlias can be anything you'd like, but it must be referenced inside of our pod's ENV variables.
		* Inside of secretObjects, we should include our secret to be synced (master-secret in our case), along with the objectName and key. 

11. Create a DNS record for our Jenkins Master. 
* DNS record will be an A record
* Make sure to check the Alias value to "yes"
* Simple routing policy
* Route it to our Network Load Balancer (NLB)

After these steps are done, make sure to change the hostname value in your Ingress to the address of your new DNS record!
 
12. Create a basic Makefile that allows for creation/stoppage of our Jenkins master and namespace with a simple command.

13. Create a basic Jenkinsfile for your test job to make sure that your Jenkins Master spins up an agent pod!
* The template we use will just determine our pod template and the workspace that we want to run the job in.

14. Deploy your yaml files and verify that your pods are running. If they are, exec into your created pod and verify that it contains the expected usernames and passwords as environment variables. 
* To do this, exec into your new pod and echo $VARIABLE_NAME to make sure that your secrets have properly been injected.

15. Let's run a test job to verify that everything works right!
* Log into Jenkins with either admin user or build user
* Click Create New Job
* Enter a name for the test job and click Pipeline -> Ok
* Enter a description and use your Jenkinsfile syntax for your test job
* Click Build Now -> Console Output to view your test job
* You can also run command ```kubectl get po -n jenkinsmaster --watch``` along side to watch the new agent pods get spun up.

## Debugging:

* Secret would not be created through SecretProviderClass
	* This was solved by adding the argument ```syncSecret.enabled=true``` to the end of the csi driver installation command.
	* If csi driver is already installed and you think you might be missing this addition, simply run the original command from the beginning of the task, but instead use: ```helm upgrade```, along with the syncSecret addition.

* Issues with key/objectName inside of SecretProviderClass
	* Your ```objectName``` should be the path/name of the secret that you created in Parameter Store. ```objectAlias``` can be named anything, but it will need to match your ENV variable ```key``` inside of your pod. 
	* Just be careful and pay close attention to where values need to be in this section. I got very confused with this part.

* Error ```exec /usr/bin/tini: exec format error```
	* This error means that your docker image was built on the wrong architecture. To fix this, rebuild your docker image with the ```--platform linux/amd64``` extension. This issue is most common if building from MacBook with M1 chip.

## Library:

Help with the setup of your Docker image and some deeper explanation
* [https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-setup-with-docker-and-jenkins-configuration-as-code](https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-setup-with-docker-and-jenkins-configuration-as-code) 
Helpful article on the steps needed for this project
* [https://docs.aws.amazon.com/systems-manager/latest/userguide/integrating_csi_driver.html#integrating_csi_driver_update_deployment](https://docs.aws.amazon.com/systems-manager/latest/userguide/integrating_csi_driver.html#integrating_csi_driver_update_deployment)
How to set up your StatefulSet to integrate with your SecretProviderClass
* [https://secrets-store-csi-driver.sigs.k8s.io/topics/set-as-env-var.html](https://secrets-store-csi-driver.sigs.k8s.io/topics/set-as-env-var.html)
More information on Pipelines with Jenkins
* [https://www.jenkins.io/doc/book/pipeline/](https://www.jenkins.io/doc/book/pipeline/)
