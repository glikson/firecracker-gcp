# Running Firecracker VMM on GCP with nested KVM

At the AWS re:Invent 2018 conference, Amazon has [open-sourced](https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/) the KVM-based virtualization runtime ([Firetracker](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md#appendix-a-setting-up-kvm-access)) they use for serverless workloads (Lambda and Fargate), which quickly became one of the most trending repositories on github.
![firecracker-trend](images/firecracker-trend.png "Firecracker trending on Github")

So, it is natural of an average geek to be curious enough to try it out. Unfortunately, currently Firecracker only runs with KVM, meaning that one would not be able to try it out in a Docker container, or even on an EC2 instance. Fortunately, Google Compute Engine (GCE) service on GCP supports nested KVM virtualization. Furthermore, if you create a GCP account, you can get $300 credits for a 1-year trial - which should be more than enough for Firetracker experiments (as well as lots of other experiments, or maybe even to run a small serverless production workload for a year).

Here are the steps to create such a setup, comprising a Ubuntu-based VM on GCE, nested KVM enablement (as documented in GCE [documentation](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances)), as well as Firecracker (following the 'getting started' [instructions](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md#appendix-a-setting-up-kvm-access)).

 1. Log in to GCP [console](https://console.cloud.google.com/) with your Google credentials. If you don't have account, you will be prompted to join the trial and get $300 credits (you do need to provide a credit card).
 
  1. Create a new project, so that you can keep track of resources more easily and remove everything easily once your are done. For convenience, give it a unique name (e.g., <your-username>-firecracker), so that GCP does not append randomized numbers to it to make it unique.
  
