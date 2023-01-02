# EKSCTL Example Configurations

[eksctl](https://eksctl.io) - The official CLI for Amazon EKS

This repository contains various examples of configurations useful for setting up EKS on AWS via [eksctl](https://eksctl.io)

eksctl is a simple CLI tool for creating and managing clusters on EKS - Amazon's managed Kubernetes service. EKSCTL basically generates and manages CloudFormation, AWS, and speaks to the Kubernetes API as well to help manage the lifecycle of things inside of Kubernetes automatically as well (such as draining nodes gradually and gracefully).


## Articles and Additional Information

I've written various articles that reference this repository, they're linked below

* [An introduction on why I recommend to use EKSCTL](https://www.devops-nirvana.com/manage-kubernetes-clusters-on-aws-with-eksctl/)
* [How to create an AWS EKS cluster via EKSCTL](https://www.devops-nirvana.com/creating-a-kubernetes-cluster-on-aws-with-eksctl/)


## Requirements

- [Install EKSCTL](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Install Kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
- [Configure your AWS CLI to have access to your AWS Account](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)


## Information on files and techniques

Herein is various EKSCTL config files that are very DRY and easy to use. They leverage YAML Aliases to copy/paste stanzas from
one node group to another. Note: Because Amazon's disks are AZ-specific, it is critical that you always create different node
groups one for every AZ. This is why every example in this repo does this. And using the YAML Alias trick this is very easy
to create two of the same and still keep the configuration very DRY and easy to edit/update. The files included are...

`gp3-best-practice.storageclass.yaml` - An alternate storageclass to the default one AWS provides, that is "best practice" in every way, not related to EKSCTL however, it is sort-of. See below for usage and see in the file for more info in the comments.

`ondemand-existing-vpc.yaml` - Creates an on-demand node group'd cluster with an existing VPC. Typically you'd create this VPC via Terraform.

`spot-existing-vpc.yaml` - Creates an spot node group'd cluster with an existing VPC. Typically you'd create this VPC via Terraform. It is
important for me to explain to you how and why the spot is configured how it is. First, because of a bug in cluster-autoscaler, you always
want your minimum nodes of a spot group to be 1, otherwise there's a bug in cluster-autoscaler being unable to scale up from 0. Secondarily,
note the price I choose for the spot helps make the spots last a long time, often 3-4 months. If you set the price you'll pay to just above the
on-demand price it enables this. Neat eh?

`existing-cluster.yaml` - This shows how you can begin to manage any existing AWS EKS cluster with EKSCTL *immediately* without having to have
set it up with EKSCTL. The key here is you need to specify the existing VPC, Subnets and Security Groups, then you can begin to manage the nodes
in that cluster. Typically using a config like this you would roll out some new nodes which can/will be managed from EKSCTL from this point forward.
After those nodes are fully live, you'd (manually of course, since you manually set it up elsewhere) drain all your old nodes one at a time and
wait for all your pods and traffic to successfully move onto your now-eksctl-managed nodes. When everything looks good, you can delete your old nodes and your old autoscalers (if enabled) and old node groups. From this point on you can easily create new nodes and drain your old ones (created by eksctl).

`simple.yaml` - Creates an VPC and a new cluster from scratch. This is the "easiest" way to on-board into using EKSCTL, requires you do
no previous setup or configuration in your AWS account. Just get up and go.

`higher-pod-density.yaml` - Creates an VPC and a new cluster from scratch (same as `simple.yaml` above), while enabling higher pod density with a new VPC CNI feature to minimize the cost of running your services if your microservices take very little CPU/RAM Resources

To use these files, see below for instructions...


## EKSCTL Usage, Hints, Tips, etc.

### Initial Run
To fully create a cluster, first configure a config file then...
```bash
# First, run the creation of the cluster, Note: replace dev.yaml with whatever file(s) from above or create your own.
# I recommend naming the file the same name as the cluster therein
eksctl create cluster --config-file dev.yaml
```

This above command fully creates the cluster, if you didn't specify an existing VPC it would also create you an VPC. It'll then
configure your Kubectl CLI so you can instantly begin to run kubectl commands. To verify you can communicate with Kubernetes, try a
sample command such as...

```bash
# Lets list the nodes in the cluster to test kubectl and your connection to it.
kubectl get nodes
```

### Setup kubectl (manually)
Incase it didn't set up your kubectl, or if you want to use it from a different machine or to let your developers access it, it's important
to know how to make your CLI setup to be able to communicate to EKS. First, you'll need the IAM Permission "eks:ListClusters" and "eks:DescribeCluster" to perform the following commands. Ask an DevOps/Sysadmin to set this up if you don't yet.
```bash
# First, show the clusters available. NOTE: If you don't see any clusters, you may need to set your region first. You can
# temporarily set your region via `export AWS_DEFAULT_REGION=us-west-1` then re-running the below command
aws eks list-clusters
{
    "clusters": [
        "dev"
    ]
}
# Hopefully you see at least one of your clusters in the command above, if that works then setup your kubectl with...
aws eks update-kubeconfig --name dev # <-- note, copy the "name" of your cluster from the above command result here
# If this works, you should be able to run...
kubectl get nodes
# or
kubectl get pods
# without an error
```

## Other Documentation / Setup / Maintenance

### If you wish to Add/Remove an managed role to the instance nodes

Sometimes you wish you added a certain managed role to your node groups, you can do this after the fact fairly easily, see the example below...

```bash
# First describe the stacks and get the instance roles
eksctl utils describe-stacks dev | grep -i NodeInstanceRole
# Then attach a new role (eg: SSM) for each nodegroup, so that aws admins can ssh into the nodes via AWS SSM (do this once per nodegroup)
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore --role-name eksctl-dev-nodegroup-c5-all-4xlar-NodeInstanceRole-ZBE0RGVPXVF6
# ^^ REPEAT ONCE FOR EVERY NodeInstanceRole

# WARNING: IF YOU DO THIS, YOUR CLOUDFORMATION WILL FAIL TO REMOVE IF YOU WISH TO ROTATE NODES OR REMOVE KUBERNETES ENTIRELY
#          To fix this, before you "delete nodegroup" or delete your cloudformation stacks you'll need to detach this above policy, eg:
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore --role-name eksctl-dev-nodegroup-c5-all-4xlar-NodeInstanceRole-XWZBXB42716V
```

#### Configure reasonable storage classes which support resizing and will retain if deleted

The default "gp2" in EKS sucks, it doesn't support volume resizing, nor will it retain your disk, plus gp2 is far inferior compared to gp3, so...

```bash
# Delete the default bad one
kubectl delete storageclass gp2
# Then install the AWS EBS CSI Driver, and create a new "default" that has gp3
kubectl apply -f gp3-best-practice.storageclass.yaml
```

#### Update all internal/default EKS utils
The first thing after a fresh install, is to update all the critical internal tools of the cluster. To make sure you
install the latest version of these tools, make SURE you have the latest release of eksctl.io installed on your computer
```bash
# Of course, replace the cluster name with the name of your cluster
eksctl utils update-aws-node --cluster dev --approve
eksctl utils update-kube-proxy --cluster dev --approve
eksctl utils update-coredns --cluster dev --approve
```


#### Upgrade CNI to latest
The very next thing you do after creating your EKS cluster before using it is update the CNI driver. This is mission critical for reliable pod IP addressing and pod security group support, amongst many other things. First, check releases: https://github.com/aws/amazon-vpc-cni-k8s/releases and grab the latest release. Then apply it (see below).
```bash
# Latest as of January 2, 2022
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/v1.12.0/config/master/aws-k8s-cni.yaml
```

After installing the CNI, there's various features I recommend enabling/configuring and they are as follows...
```bash
# Enable the CNI plugin to manage network interfaces for pods
kubectl set env daemonset aws-node -n kube-system ENABLE_POD_ENI=true
# Allow liveness probes to succeed (with a SG attached to it)
kubectl patch daemonset aws-node \
  -n kube-system \
  -p '{"spec": {"template": {"spec": {"initContainers": [{"env":[{"name":"DISABLE_TCP_EARLY_DEMUX","value":"true"}],"name":"aws-vpc-cni-init"}]}}}}'
# Enable more IP addresses to be set per-node (meaning more services with SGs allowed per-node), a new feature in 2022
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
```

#### PAUSE / NOTE
After doing some all of the setup above you may need to rotate all nodes to get full functionality, especially the pod security group feature working. Feel free to try some things to make sure they work if needed before continuing to setup and provision Kubernetes services.

### Creating New Node Groups and/or Rotating out old ones
Begin, by creating a new node group to the <envname>.yaml and get it reviewed. Then, to apply changes in this config run...
```bash
# Add new node groups that you added to the file...
eksctl create nodegroup --config-file=dev.yaml
# Remove old node groups (preview what it would do)  (NOTE: replace the "include" name to be the name of the nodegroup you are deleting)
eksctl delete nodegroup --config-file=./dev.yaml  --include "k8s-2b-c6g-xlarge-primary"
# Remove old node groups (ACTUALLY do it)
eksctl delete nodegroup --config-file=./dev.yaml  --include "k8s-2b-c6g-xlarge-primary" --approve
```

### If you wish for Pod Security Group / IRSA (IAM Roles for Service Accounts) support and didn't enable in the YAML
```bash
# Add oidc after incase it didn't add, note this is now supported to be in the yaml
# but you can run this incase you manage an existing cluster with EKSCTL and don't have this enabled
eksctl utils associate-iam-oidc-provider --cluster "dev" --approve
```

### To change logging settings...
```bash
eksctl utils update-cluster-logging --enable-types=authenticator,api  --disable-types=audit,controllerManager,scheduler --cluster CLUSTERNAME --approve
```
