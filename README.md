# task2

# Create a service account
1. Create the service account itself:

```console
gcloud iam service-accounts create jenkins --display-name jenkins
```

2. Store the service account email address and your current Google Cloud Platform (GCP) project ID in environment variables for use in later commands:
```console
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:jenkins" --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Bind the following roles to your service account:
```console
gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/iam.serviceAccountActor \
    --member serviceAccount:$SA_EMAIL
```
# Download the service account key
Now that you've granted the service account the appropriate permissions, you need to create and download its key. Keep the key in a safe place. You'll use it later step when you configure the JClouds plugin to authenticate with the Compute Engine API.

1. Create the key file:
```console
gcloud iam service-accounts keys create jenkins-sa.json --iam-account $SA_EMAIL
```
2. In Cloud Shell, click More :, and then click Download file.

3. Type jenkins-sa.json.

4. Click Download to save the file locally.

# Create a Jenkins agent image
Next, you create a reusable Compute Engine image that contains the software and tools needed to run as a Jenkins executor.

Create an SSH key for Cloud Shell
Use Packer to build your images, which requires the ssh command to communicate with your build instances. To enable SSH access, create and upload an SSH key in Cloud Shell:

Create a SSH key pair. If one already exists, this command uses that key pair; otherwise, it creates a new one:
```console
ls ~/.ssh/id_rsa.pub || ssh-keygen -N ""
```
Add the Cloud Shell public SSH key to your project's metadata:
```console
gcloud compute project-info describe \
    --format=json | jq -r '.commonInstanceMetadata.items[] | select(.key == "ssh-keys") | .value' > sshKeys.pub
echo "$USER:$(cat ~/.ssh/id_rsa.pub)" >> sshKeys.pub
gcloud compute project-info add-metadata --metadata-from-file ssh-keys=sshKeys.pub
```
Create the baseline image

The next step is to use Packer to create a baseline virtual machine (VM) image for your build agents, which act as ephemeral build executors in Jenkins. The most basic Jenkins agent only requires Java to be installed. You can customize your image by adding shell commands in the provisioners section of the Packer configuration or by adding other Packer provisioners.

In Cloud Shell, download and unpack Packer:
```console
wget https://releases.hashicorp.com/packer/0.12.3/packer_0.12.3_linux_amd64.zip
unzip packer_0.12.3_linux_amd64.zip
```
Create the configuration file for your Packer image builds:
```console
export PROJECT=$(gcloud info --format='value(config.project)')
cat > jenkins-agent.json <<EOF
{
  "builders": [
    {
      "type": "googlecompute",
      "project_id": "$PROJECT",
      "source_image_family": "ubuntu-1604-lts",
      "source_image_project_id": "ubuntu-os-cloud",
      "zone": "us-central1-a",
      "disk_size": "10",
      "image_name": "jenkins-agent-{{timestamp}}",
      "image_family": "jenkins-agent",
      "ssh_username": "ubuntu"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": ["sudo apt-get update",
                  "sudo apt-get install -y default-jdk"]
    }
  ]
}
EOF
```
Build the image by running Packer:
```console
./packer build jenkins-agent.json
```
When the build completes, the name of the disk image is displayed with the format jenkins-agent-[TIMESTAMP], where [TIMESTAMP] is the epoch time when the build started.
```console
==> Builds finished. The artifacts of successful builds are:
--> googlecompute: A disk image was created: jenkins-agent-{timestamp}
```

# Now we will create an Jenkins instance via terraform .

Clone the repository https://github.com/somud17/task2
```console
git clone https://github.com/somud17/task2
````

Create jenkins instance with ready kubernetes cluster, follow below steps:
```console
terraform init
terraform plan
terraform validate
terraform apply

```
