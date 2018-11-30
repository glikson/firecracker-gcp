# Running Firecracker VMM on GCP with nested KVM

At the AWS re:Invent 2018 conference, Amazon has [open-sourced](https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/) the KVM-based virtualization runtime ([Firecracker](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md#appendix-a-setting-up-kvm-access)) they use for serverless workloads (Lambda and Fargate), which quickly became one of the most trending repositories on github.

So, it is natural for an average geek to be curious enough to try it out. Unfortunately, Firecracker currently only works with KVM, meaning that one would not be able to try it out in a Docker container on their laptop, or even on an EC2 instance. Fortunately, Google Compute Engine (GCE) supports nested KVM virtualization. Furthermore, if you create a GCP account, you can get $300 credits for a 1-year trial - which should be more than enough for Firetracker experiments (as well as lots of other experiments, or maybe even to run a small serverless production workload for a year).

Here is a brief summary of steps to create such a setup, comprising a Ubuntu-based VM on GCE, nested KVM enablement (as documented in GCE [documentation](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances)), as well as Firecracker (following the 'getting started' [instructions](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md)).

  1. Log in to GCP [console](https://console.cloud.google.com/) with your Google credentials. If you don't have account, you will be prompted to join the trial and get $300 credits (you do need to provide a credit card).
 
  1. Create a new project, so that you can keep track of resources more easily and remove everything easily once your are done. For convenience, give the project a unique name (e.g., <your-username>-firecracker), so that GCP does not need to create project id different than project name (by appending randomized numbers to the name you provide).
  
  1. You would need the `gcloud` CLI installed and configured to interact with your project on GCP. If you don't have it yet, here is how to do it on an EC2 t2.micro instance running Ubuntu 16.  
     1. Install GCP CLI & SDK (full instructions can be found [here](https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu))
        ```
        $ export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
        $ echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
        $ curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        $ sudo apt-get update && sudo apt-get install -y google-cloud-sdk
        ```        
     1. Configure the CLI by running:
        ```
        $ gcloud init --console-only
        ```
        Follow the prompts to authenticate (open the provided link, authenticate, copy the token back to console) and select the project you created
     1. Alternatively, if your `gcloud` CLI is already configured, just switch to the new project using:
        ```
        $ gcloud config set project <your-project>
        ```
  1. The next step is to create a VM image able to run nested KVM (as outlined [here](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances)). **IMPORTANT:** Notice that Firecracker requires a relatively new kernel, so you should use a recent Ubuntu 18 image (or equivalent).
     ```
     $ gcloud compute disks create disk-ub18 --image-project ubuntu-os-cloud --image-family ubuntu-1804-lts
     ```
     If you are prompted whether to enable the Compute API for this project, confirm, wait until prompted again, select region - e.g., us-east1-b
     ```
     $ gcloud compute images create ub18-nested-kvm --source-disk disk-ub18 --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" --source-disk-zone us-east1-b
     ```
     Make sure you run it as a single command, and that you specify the right region.
  1. Now we create the VM:
     ```
     $ gcloud compute instances create firecracker-vm --zone us-east1-b --image ub18-nested-kvm
     ```
  1. Connect to the VM via SSH.  
     ```
     $ gcloud compute ssh firecracker-vm
     ```
     When doing it for the first time, a key-pair will be created for you (you will be propmpted for a passphrase - can just keep it empty) and uploaded to GCE. Done! You should see the prompt of the new VM: `ubuntu@firecracker-vm:~$`  
  1. Verify that VMX is enabled, install and enable KVM
     ```
     $ grep -cw vmx /proc/cpuinfo
     1
     $ sudo apt-get update && sudo apt-get install qemu-kvm -y
     $ sudo setfacl -m u:${USER}:rw /dev/kvm
     $ [ -r /dev/kvm ] && [ -w /dev/kvm ] && echo "OK" || echo "FAIL"
     OK
     ```   
  1. Download Firecracker (based on instructions [here](https://github.com/firecracker-microvm/firecracker/blob/master/docs/getting-started.md)). Optionally change the first URL below to the release you want, from https://github.com/firecracker-microvm/firecracker/releases
     ```
     $ curl -fsSL -o ./firecracker https://github.com/firecracker-microvm/firecracker/releases/download/v0.11.0/firecracker-v0.11.0
     $ chmod +x ./firecracker
     $ curl -fsSL -o ./hello-vmlinux.bin https://s3.amazonaws.com/spec.ccfc.min/img/hello/kernel/hello-vmlinux.bin
     $ curl -fsSL -o ./hello-rootfs.ext4 https://s3.amazonaws.com/spec.ccfc.min/img/hello/fsfiles/hello-rootfs.ext4
     ```
  1. Run Firecracker
     1. Run the VMM (if everything is working, it will NOT return to your command prompt):
        ```
        $ rm -f /tmp/firecracker.sock && ./firecracker --api-sock /tmp/firecracker.sock
        ```
     1. Open an additional terminal, SSH into your `firecracker-vm`. Configure and start the Guest VM (ignore the 'No Content' outputs):
        ```
        $ curl --unix-socket /tmp/firecracker.sock -i \
          -X PUT 'http://localhost/boot-source' -H 'Accept: application/json' -H 'Content-Type: application/json' \
          -d '{"kernel_image_path":"./hello-vmlinux.bin","boot_args":"console=ttyS0 reboot=k panic=1 pci=off"}'
        $ curl --unix-socket /tmp/firecracker.sock -i \
          -X PUT 'http://localhost/drives/rootfs' -H 'Accept: application/json' -H 'Content-Type: application/json' \
          -d '{"drive_id":"rootfs","path_on_host":"./hello-rootfs.ext4","is_root_device":true,"is_read_only":false}'
        $ curl --unix-socket /tmp/firecracker.sock -i \
          -X PUT 'http://localhost/actions' -H  'Accept: application/json' -H  'Content-Type: application/json' \
          -d '{"action_type":"InstanceStart"}'
        ```
     1. Switch back to the **first** terminal (where VMM is running), use 'root/root' to log into the guest. **Done!**
        ```
        Welcome to Alpine Linux 3.8
        Kernel 4.14.55-84.37.amzn2.x86_64 on an x86_64 (ttyS0)
        
        localhost login: root
        Password:
        Welcome to Alpine!
        
        The Alpine Wiki contains a large amount of how-to guides and general
        information about administrating Alpine systems.
        See <http://wiki.alpinelinux.org>.
        
        You can setup the system with the command: setup-alpine
        
        You may change this message by editing /etc/motd.
        
        login[858]: root login on 'ttyS0'
        localhost:~#
        ```
**WARNING:** Keep in mind that Firecracker is very new, so the above instructions might become obsolete. Also, notice that this is not even a full 'hello world' example, given that guest VM is pretty much useless without network etc.
