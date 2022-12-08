# Creating an Kubernetes cluster on AWS with EKSCTL

Creating an Kubernetes cluster on Amazon can seem to be a simple matter, many methods and tools exist now to support and simplify this process, however most tools fall flat when it comes to their usage of supporting an EKS cluster over time, through various upgrades in both the version of EKS and in replacing node groups with zero-downtime.  However, your life can be made easy if you know the tool and method(s) I use and recommend.


## The methods of creating Kubernetes clusters

So first, lets go over the options that I know off the top of my head of how to create and manage Kubernetes clusters on AWS

* You can setup your own self-manager Kubernetes cluster from scratch
* You can use a tool called [KOPS](#TODO) to assist you in creating an self-managed AWS Cluster
* You can use the AWS Console to create an AWS-managed cluster
* You can use a [Terraform module](#TODO) to assist creating an AWS-managed Cluster
* You can use a [Ansible module](#TODO) to assist creating an AWS-managed Cluster
* You can use a [Pulumi module](#TODO) to assist creating an AWS-managed Cluster
* You can use an tool co-authored by AWS and TODOCOMPANY called EKSCTL

Now lets discuss the options above and their upsides and downsides.

If you go with either of the self-managed options (the first two options), you and your team are in for a lot of work and a lot of unnecessary tech debt.  Self-managed Kubernetes clusters means that you run and support the "master" nodes (for more info about master Kubernetes nodes [click here](#TODO) ).

One of the core-tenets of successful DevOps is not taking on responsibility for something that you don't need to.  So to succeed with that aim, we won't even consider or think about the first two further.  However, I think it's important for any fellow professional to know what KOPS is, as this was "the way" to deploy Kubernetes before Amazon created EKS.

Next, is all the various options/ways to spin up an AWS managed EKS cluster.  This includes doing it manually via the AWS Console, or automatically with any various set(s) of tools.  The model I typically follow for my clients is create everything via Terraform (eg: VPC, subnets, NAT Gateway, etc) EXCEPT for EKS, and for EKS you use the tool Amazon helped co-author called EKSCTL.  Generally, for good DevOps practices you NEVER want anything manually setup, and because EKSCTL only configures EKS you'd need some framework around where EKS will go into (eg: the VPC).  However, if you're just starting/learning, maybe you are just trying to test out EKS, you might want to start with just letting EKSCTL create your VPC for you also.


## Why EKSCTL

Now it's important to understand WHY I have chosen EKSCTL over any other option.

EKS is a fairly complex beast to setup and that setup changes over time.  This is the ONE thing that I do not recommend setting up and managing via Terraform, but instead with a tool written by and recommend by AWS called [EKSCTL](https://eksctl.io/).  For longer-term usage and support of EKS, I have encountered just about every issue you could imagine with every one of those methods above, the biggest problem with all of the other methods is that they don't manage the lifecycle of EKS.  They don't "speak" EKS, and they don't understand how to manage and maintain an EKS cluster with zero-downtime over time.  I personally have encountered various issues with all other methods, which will only surface after you've been using them a while and need to perform version updates and/or node upgrades.  My current recommended path to success for EKS is EKSCTL from Amazon.  I've been successfully setup and maintained and upgraded more than 30 EKS clusters via EKSCTL over the last 3 (YEAR DATE ME) years, and I have struggled with supporting yet another 20+ EKS clusters via either the AWS Console directly, or via Terraform.

Largely, what happens a year down the road is you need to upgrade EKS.  With EKSCTL, it's just a series of steps and commands to EKSCTL to perform zero-downtime rollouts of new node groups.  The reason EKSCTL is better in this regard, is it "speaks" to Kubernetes and manages the lifecycle of events.  When you destroy a node group in Terraform, it would just go destroy an autoscaling group and kill the underlying nodes, causing pods to suddenly be forced off their node, and potentially causing downtime beause of the unexpected/unplanned nature of nodes being terminated.  With EKSCTL, when you want to remove a node group, it goes into Kubernetes and one-at-a-time cordons and drains the nodes (explain what this is) until all nodes from this autoscaler are drained, then it removes the nodes and then removes the autoscaler.  This method of graceful incremental removal of nodes helps manage and reduce or eliminate downtime entirely (as long as you've configured your pods for high-scalability/redundancy).

This incremental node replacement strategy is part of something Amazon tried to manage for you called "Managed node groups", and while you are welcome to use that, I personally don't recommend it.  When you push off this concern into Amazon, you can find the removal of node groups to "stick" because of PDBs (explain) and because this managed node group is a bit of a "black box" you don't really get feedback from it.  When you use a tool like EKSCTL to manage this, you get the full information of what is happening, and if there's any hiccups or of your removal of node groups gets "stuck", you'll know right away and you can manage it yourself how you want.  This allows you the ability to, say, go scale up a service suddenly so it lands on some of the other nodes, before then removing the pod(s) on the node you are trying to delete.  This concept is simply not possible when you let AWS manage the node group replacement.  Additionally, Amazon's managed node groups has some limitations, namely that it [doesn't support Windows](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html) and that you don't have the same amound of flexibility in the way you configure its spot instance capabilities (allows for multi-spot instance type choices).

It says:
You can't deploy Windows managed nodes. You can only create self-managed Windows nodes. For more information, see [Launching self-managed Windows nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-windows-workers.html).


## Requirements / Setup

- [Install EKSCTL](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Install Kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
- [Configure your AWS CLI to have access to your AWS Account](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

## Get started making your cluster

You have really one of two options, you either pre-create (or use an existing) VPC on AWS, OR you let EKSCTL create that VPC.

For simplicity in this post we will be letting EKSCTL create an VPC.  Using your favorite editor, just create a file named `basic-cluster.yaml` with the following contents.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: us-east-1
  version: "1.23"
iam:
  withOIDC: true

nodeGroups:
  - name: ng-1
    availabilityZones: ["us-east-1a"]
    instanceType: m5.large
    desiredCapacity: 10
    volumeSize: 50
  - name: ng-2
    availabilityZones: ["us-east-1b"]
    instanceType: m5.xlarge
    desiredCapacity: 2
    volumeSize: 50
```

Of course, change the region, availability zones, and cluster name, and version if desired.  Save this file, and then run the command

```
eksctl create cluster --config-file basic-cluster.yaml
```

After running this command, it will begin provisioning your "infrastructure as code" eksctl config file in AWS.  EKSCTL does this by creating some  [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) based on the configuration file.  Once your cluster has been provisioned, it will then provision the node groups.  After it finishes with the node groups, it'll setup your `kubectl` current context to be talking to the cluster, and it will exit.

At this point you should be able to query your Kubernetes cluster with the command

```bash
kubectl get nodes
```

If this works, then congratulations you've bootstrapped your own Kubernetes cluster on AWS via EKSCTL.

Stay tuned for some of the follow-up articles on this regarding how to roll out new nodes, and on how to update Kubernetes versions with zero-downtime!

For much more useful and detailed EKSCTL configurations, I recommend viewing our companion Github repository with EKSCTL example configurations.  This will fit much more advanced scenarios, spot instances, etc.
https://github.com/DevOps-Nirvana/EKSCTL-Example-Configurations
