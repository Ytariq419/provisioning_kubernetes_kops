**Installing Kubernetes With KOPS**

**Before you begin**

You must have kubectl installed. You must install kops. You must have an AWS account, generate IAM keys and configure them. The IAM user will need adequate permissions.
	

**Creating a cluster**

Install kops

Download kops from the releases page (it is also convenient to build from source):


Download the latest release with the command:

For Mac:
```

curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -
	
```

You can also install kops using Homebrew.
```
brew update && brew install kops
```

**Create a route53 domain for your cluster**

Kops uses DNS for discovery, both inside the cluster and outside, so that you can reach the kubernetes API server from clients. Kops has a strong opinion on the cluster name: it should be a valid DNS name. By doing so you will no longer get your clusters confused, you can share 	clusters with your colleagues unambiguously, and you can reach them without relying on remembering an IP address.

You can, and probably should, use subdomains to divide your clusters. As our example we will use useast1.dev.example.com. The API server endpoint will then be api.useast1.dev.example.com.

A Route53 hosted zone can serve subdomains. Your hosted zone could be useast1.dev.example.com, but also dev.example.com or even example.com. kops works with any of these, so typically you choose for organization reasons (e.g. you are allowed to create records under dev.example.com, but not under example.com).

Let's assume you're using dev.example.com as your hosted zone. You create that hosted zone using the normal process, or with a command such as aws route53 create-hosted-zone --name dev.example.com --caller-reference 1.

You must then set up your NS records in the parent domain, so that records in the domain will resolve. Here, you would create NS records in example.com for dev. If it is a root domain name you would configure the NS records at your domain registrar (e.g. example.com would need to be configured where you bought example.com).

Verify your route53 domain setup (it is the #1 cause of problems!). You can double-check that your cluster is configured correctly if you have the dig tool by running:
```
dig NS dev.example.com

You should see the 4 NS records that Route53 assigned your hosted zone.
```


**Create an S3 bucket to store your clusters state**

Kops lets you manage your clusters even after installation. To do this, it must keep track of the clusters that you have created, along with their configuration, the keys they are using etc. This information is stored in an S3 bucket. S3 permissions are used to control access to the bucket.

Multiple clusters can use the same S3 bucket, and you can share an S3 bucket between your colleagues that administer the same clusters - this is much easier than passing around kubecfg files. But anyone with access to the S3 bucket will have administrative access to all your clusters, so you don't want to share it beyond the operations team.
So typically you have one S3 bucket for each ops team (and often the name will correspond to the name of the hosted zone above!)

In our example, we chose dev.example.com as our hosted zone, so let's pick clusters.dev.example.com as the S3 bucket name.

Export AWS_PROFILE (if you need to select a profile for the AWS CLI to work) 

Create the S3 bucket using: 
```
aws s3 mb s3://clusters.dev.example.com 
```

You can export ```KOPS_STATE_STORE=s3://clusters.dev.example.com``` and then kops will use this location by default. We suggest putting this in your bash profile or similar. 


**Build your cluster configuration**

Run kops create cluster to create your cluster configuration:
```
kops create cluster --zones=us-east-1c useast1.dev.example.com
```

kops will create the configuration for your cluster. Note that it only creates the configuration, it does not actually create the cloud resources - you'll do that in the next step with a ```kops update cluster```. This give you an opportunity to review the configuration or change it. It prints commands you can use to explore further:
	List your clusters with: ```kops get cluster```
	Edit this cluster with: ```kops edit cluster useast1.dev.example.com```
	Edit your node instance group:``` kops edit ig --name=useast1.dev.example.com nodes```
	Edit your master instance group:``` kops edit ig --name=useast1.dev.example.com master-us-east-1c```


**Create the cluster in AWS**

Run "kops update cluster" to create your cluster in AWS:
```
kops update cluster useast1.dev.example.com --yes
```

That takes a few seconds to run, but then your cluster will likely take a few minutes to actually be ready. ```kops update cluster``` will be the tool you'll use whenever you change the configuration of your cluster; it applies the changes you have made to the configuration to your cluster - reconfiguring AWS or kubernetes as needed.

For example, after you ```kops edit ig nodes```, then ```kops update cluster --yes``` to apply your configuration, and sometimes you will also have to kops rolling-update cluster to roll out the configuration immediately.
Without --yes, kops update cluster will show you a preview of what it is going to do. This is handy for production clusters!

**Cleanup**
To delete your cluster: 
```
kops delete cluster useast1.dev.example.com --yes
```
