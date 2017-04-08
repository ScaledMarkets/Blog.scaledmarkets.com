---
layout: post
title: Creating a Kubernetes Cluster With kube-aws 
---

#Creating a Kubernetes Cluster With kube-aws

Kubernetes is one of the most promising container orchestration frameworks, because it is portable—not tied to a particular cloud provider (even though Google created it)—and because it is full-featured.

Unfortunately, creating a Kubernetes cluster is a little bit complex. The Kubernetes project currently provides two ways to create a cluster: (1) the kube-up script, which can be accessed from https://get.k8s.io, and (2) the kops tool, which has great potential but is relatively immature at this point. You can also create a cluster by hand, if you know what you are doing. But if you are working in AWS, then the best way currently, in my opinion, is by using the CoreOS “kube-aws” tool.
What Is a Kubernetes Cluster?
A cluster is just a collection of machines that are managed by a Kubernetes cluster controller. The machines can be real machines or virtual machines (VMs). Using real machines is more efficient since one then does not have the overhead of a hypervisor, which is redundant when using containers. (It remains to be seen what will become of the large commercial hypervisor-based VM ecosystem that exists today.)

If you are a software developer in a large organization, you will likely already have a cluster to deploy to—it would have been created for you. Alternatively, some teams prefer to create a cluster for each application, or for groups of applications. Regardless, someone has to create the cluster, and if you are working on your own, or in a self-sufficient team, then you do.

As I said above, I recommend the CoreOS kube-aws tool for creating Kubernetes clusters. Instructions for obtaining and using kube-aws can be found here. However, those instructions, while excellent overall, leave a few details out. Below I provide some tips that will hopefully fill in the gaps.
AWS Steps
I generally do all of my command line work on an AWS node, and I recommend that you do that too for setting up a Kubernetes cluster in AWS. Create a Linux VM in AWS, ssh into that instance, and then perform the steps described in the kube-aws instructions, with the tips below.
Obtain kube-aws
Tip: To see the full list of options for gpg2, type,
gpg2 --dump-options

Tip: To download kube-aws using curl, use the -L and -O options:
curl -L -O https://github.com/coreos/coreos-kubernetes/releases/download/v0.8.2/kube-aws-linux-amd64.tar.gz
Configure AWS Access
Tip: The AWS command line tools will have to be installed on your system. See here if you need to install them.

Tip: Instead of creating ~/.aws/credentials and ~/.aws/config files, you can set these environment variables:
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION
Configure Key-Based Access to AWS Instances
Tip: Create an AWS EC2 SSH key pair, if you do not already have one. You give the key pair a name when you create it, and the public key is retained by AWS. You can then access it via the “Key Pairs” section of the EC2 console.

When a key pair is created, you will be instructed to download the public and private key pair and store the private key securely—either in your ~/.ssh directory, or in a project-specific location, and it should have 0400 access. You will specify the name of this key pair when you configure the Kubernetes cluster, and Kubernetes will use AWS to install the public key on each machine in the cluster. The presence of that key on each machine enables you to perform ssh -i <private-key-location> <userid>@<host> to connect to the machines that were created.

Tip: Create an SSL/TLS key pair. AWS provides a tool/system called KMS for this purpose. This is used for establishing SSL/TLS connections between users of your application and your Kubernetes load balancer.

Tip: In order to create the domain name that kube-aws requires as the name of the cluster controller, you will have to have an existing DNS domain so that you can create the cluster controller’s domain as a subdomain. To create that existing domain in AWS, you can use the Route 53 service: create a “Hosted Zone”, for example mydomain.com, and then for the Kubernetes cluster controller, specify a new subdomain, such as cluster.mydomain.com. kube-aws will automatically create the subdomain.
Create Cloud Formation Assets
kube-aws works by creating an AWS Cloud Formation file. Cloud Formation is AWS’s native computing environment orchestration tool. This is a good thing because by using Cloud Formation, kube-aws benefits from the maturity and robustness of Cloud Formation.
Create Directory For the Cluster Configuration
When creating a cluster, you keep the cluster’s configuration files in a directory on the machine from which you plan to administer the cluster. This is similar to other environment creation tools such as vagrant. Create a directory for your cluster:

mkdir mycluster
cd mycluster
Create Configuration File
Tip: If you want kube-aws to create the subdomain cluster.mydomain.com off of the existing domain mydomain.com, your AWS region is us-west-2, you would like the cluster to be created in zone us-west-2b of region us-west-2, your SSH key pair name is MyKeyPair (in which case, you will have downloaded a file called MyKeyPair.pem from AWS), and your SSL/TLS key (see “Encryption Keys” in the AWS “Identity and Access Management” console) ARN is arn:aws:kms:my-region:xxxxxxxxxxx:key/xxxxxxxxxxxxxxxx,

kube-aws init \
--cluster-name=mycluster \
--external-dns-name=cluster.mydomain.com \
--region=us-west-2 \
--availability-zone=us-west-2b \
--key-name=MyKeyPair \
--kms-key-arn="arn:aws:kms:my-region:xxxxxxxxxxx:key/xxxxxxxxxxxxxxxx"

Tip: The cluster-name must be unique within your AWS account.

Tip: The second level domain name of the subdomain specified by the external-dns-name—i.e., the “mydomain.com” part of cluster.mydomain.com—must be a valid registered DNS name. Merely creating a Hosted Zone in AWS for that domain will not be sufficient. You must create the hosted zone for the domain name (e.g., mydomain.com) in AWS, and also register the domain name with a domain registrar. AWS provides domain registration—see here.

Tip: Edit the resulting cluster.yaml file. E.g., set these to the values you prefer:

createRecordSet: true
hostedZoneId: <id>
workerCount: 2
workerInstanceType: m3.medium
workerRootVolumeSize: 30
tlsCADurationDays: 3650
tlsCertDurationDays: 365

where the <id> is the “Hosted Zone ID” of your AWS Route 53 hosted zone.

By default, kube-aws will create its own AWS Virtual Private Cloud (VPC). However, you can set the vpcId parameter in the cluster.yaml file if you want your cluster to use an existing VPC that you have created through other means.

Note: As of this writing, the generated cluster.yaml file uses the v1.3.6_coreos.0 version of the CoreOS image quay.io/coreos/hyperkube, which—according to CoreOS’s own scan, has 95 security vulnerabilities. You can select a different version by editing the kubernetesVersion value in the cluster.yaml file. However, even the most recent version (as of this writing) has 72 vulnerabilities. Some of the vulnerabilities involve denial of service attacks on the SSL implementation. This means that the cluster controller should only be deployed in a secure network and should not be directly accessible from the Internet—access from the Internet should only be via a proxy or a VPN/VPC.
Generate Cloud Formation
All we need to do is type,

kube-aws render
kube-aws validate

The output is a 600+ line stack-template.json file, which is a Cloud Formation file, and a kubeconfig file, which specifies which credentials to use when authenticating to your cluster.
Launch Cluster
kube-aws up

At this point, it displays a message, “Creating AWS resources. This should take around 5 minutes” - and indeed it does take that long, and gives you no feedback while it is working, but you can see progress if you go to the AWS Cloud Formation console, where you should see the “stack” listed with a “CREATE_IN_PROGRESS” message. When it finished the status should change to “CREATE_COMPLETE”.

When it completes you should also have command line output like this:

Creating AWS resources. This should take around 5 minutes.
Success! Your AWS resources have been created:
Cluster Name:    mycluster
Controller IP:    100.100.100.100

The containers that power your cluster are now being downloaded.

You should be able to access the Kubernetes API once the containers finish downloading.

If you then go to your AWS EC2 instance console, you should now see something like what is shown in the Figure below. Notice the first two instances: these are what I got when I launched a cluster for ScaledMarkets’ safeharbor service. If I had set the workerCount to be more than one, then there would be that many instances of kube-aws-worker.



Verify that the Cluster Is Accessible
You can now perform Kubernetes commands to manage your cluster. The primary command is the kubectl command. I launched my cluster on an x86-64 Centos 7 system, so I put the /kubernetes/platforms/linux/amd64 directory in my path. To verify access to the cluster, try

kube-aws status

That should give you something like this:

Cluster Name:    mycluster
Controller IP:    100.100.100.100

To print descriptions of the nodes in the cluster, use the kubectl command:

kubectl --kubeconfig=kubeconfig get nodes

Assuming that your DNS domain is registered and that there is an AWS Hosted Zone for it, you should get something like this:

NAME                                       STATUS                     AGE
ip-10-0-0-156.us-west-2.compute.internal    Ready                      2m
ip-10-0-0-50.us-west-2.compute.internal    Ready,SchedulingDisabled    2m
Deleting a Cluster
The command,

kube-aws destroy

deletes the cluster. However, it does not give you any feedback on progress. If you want to see that, go to your AWS Cloud Formation console, and you should see the Cloud Formation “stack” created by kube-aws with the message, DELETE_IN_PROGRESS.

Unfortunately, kube-aws destroy does not always work: it often fails to remove things like the VPC, load balancer, and other things that kube-aws up created. Thus, you might have to go into the Cloud Formation console event list to look at the messages and see which deletions failed, then then delete those things manually.
Using the Kubernetes Dashboard
Kubernetes has a nice dashboard for viewing and managing your cluster. However, your cluster is in AWS. If you are SSH-ing into a VM in AWS as I recommended, you will not be able to launch a browser there to view the dashboard. There are many ways to get around that. One is to boot a VM with X-Windows or Wayland and access that remotely. I have not tried that, so I can’t give you advice there. Another approach is to launch the AWS-based cluster from your local machine instead of from a VM in AWS as I have recommended. I don’t like to execute projects on my local machine because then I muck up my local machine—I like to always work from a VM that I can then blow away, using a separate VM for each project—but you can also avoid mucking up your local machine by creating a local VM with a console (“headed”) and working from that. It should work fine and be secure, but I don’t work that way. Finally, you can use run the Kubernetes proxy in AWS and access the proxy via the VM’s public IP address. To do that, type this on your AWS VM command line:

kubectl --kubeconfig kubeconfig proxy --accept-hosts="^*$" --address="<your-AWS-VM-private-IP-address>"

where <your-AWS-VM-private-IP-address> is the private IP address of your AWS VM. You can obtain that from the EC2 instance console. Then add port 8001 to the security group for your AWS VM, allowing any IP address for egress, but limiting to your local IP address for ingress—that is the “Source” setting in the AWS Security Group.

You can then view the dashboard at,

http://<your-AWS-VM-public-IP-address>:8001/ui

where <your-AWS-VM-public-IP-address> is the public IP address of your AWS VM.

The dashboard should look something like what is shown below.



You can click on the various dashboard elements to drill down—you can even edit a pod’s configuration and restart the pod, but I have not tried that. If you click on a pod, you will then see a detail page, showing the pod’s full name—the name shown on the dashboard main page is only an abbreviated name. Importantly, you can use the dashboard to find out the generated names of your pods, so that you can then perform actions such as attaching to a container:

kubectl --kubeconfig=kubeconfig attach <pod> -c <container>

Note that the kubectl proxy command blocks—canceling it with ^C will terminate the proxy—so you will probably want to run it with something like,

nohup kubectl --kubeconfig kubeconfig proxy --accept-hosts="^*$" --address="<your-AWS-VM-private-IP-address>" > log.out 2> log.err < /dev/null &

Note: Using the proxy is not a secure setup, because while you have restricted dashboard access to your IP address, IP addresses can be spoofed, and the proxy uses HTTP (not HTTPS). Therefore, you should only use the proxy from within a VPC/VPN or secure local network. However, when working in AWS I strongly recommend always working within a VPC/VPN anyway.
Conclusion
Once a cluster has been defined, we have one command to launch an entire cluster of machines, and we can administer that cluster with the kubectl command. The hard part was in creating the AWS keys and config files, but now that that has been done, we can create additional clusters with ease. Thus, we now have a repeatable and reliable process for standing up Kubernetes clusters—using AWS’s native features.

Now what? We have a cluster, but what can we do with it? That will be the topic of the next article, “Deploying a Kubernetes Service In Three Easy Steps”, in which I explain how to define and deploy an application configuration with Kubernetes.
