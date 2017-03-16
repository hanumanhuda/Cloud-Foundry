Cloud Foundry:
Cloud Foundry is an open source, multi cloud application platform as a service (PaaS) governed by the Cloud Foundry Foundation organization.
Bosh:
BOSH is an open source project which offers a tool chain for release engineering, deployment & life-cycle management of large scale distributed services. Namely, this tool chain is made of a server (the BOSH Director) and a command line tool. 
BOSH Components
bosh-init
Command Line Interface (CLI)
Director
Task Queue
Workers
Cloud Provider Interface (CPI)
Health Monitor
Resurrector
DNS Server
Components used to store Director's persistent data
Database
Blobstore
Agent
Components used for cross-component communication
Message Bus (NATS)
Registry
Example component interaction
Creating a new VM

Before bootstrapping a new environment we recommend to learn the names of major components that will be installed, used and configured:


bosh-init
bosh-init is a tool used for creating and updating the Director (its VM and persistent disk) in an environment.

Command Line Interface (CLI)
The Command Line Interface (CLI) is the primary operator interface to BOSH. An operator uses the CLI to interact with the Director and perform actions on the cloud.
CLI is typically installed on a machine that can directly communicate with the Director’s API, e.g. an operator’s laptop, or a jumpbox in the datacenter.

Director
The Director is the core t component in BOSH. The Director controls VM creation and deployment, as well as other software and service lifecycle events.
The Director creates actionable tasks:
By translating commands sent by an operator through the CLI
From scheduled processes like backups or snapshots
If needed to reconcile the expected state with the actual state of a VM
Once created, the Director adds these tasks to the Task Queue. Worker processes take tasks from the Task Queue and act on them.
Task Queue
An asynchronous queue used by the Director and Workers to manage tasks. The Task Queue resides in the Database.
Workers
Director workers take tasks from the Task Queue and act on them.
Cloud Provider Interface (CPI)
A Cloud Provider Interface (CPI) is an API that the Director uses to interact with an IaaS to create and manage stemcells, VMs, and disks. A CPI abstracts infrastructure differences from the rest of BOSH.

Health Monitor
The Health Monitor uses status and lifecycle events received from Agents to monitor the health of VMs. If the Health Monitor detects a problem with a VM, it can send an alert through notification plugins, or trigger the Resurrector.
Resurrector
If enabled, the Resurrector plugin automatically recreates VMs identified by the Health Monitor as missing or unresponsive. It uses same Director API that CLI uses.

DNS Server
BOSH uses PowerDNS to provide DNS resolution between the VMs in a deployment.

Components used to store Director’s persistent data
Database
The Director uses a Postgres database to store information about the desired state of a deployment. This includes information about stemcells, releases, and deployments.
Blobstore
The Blobstore stores the source forms of releases and the compiled images of releases. An operator uploads a release using the CLI, and the Director inserts the release into the Blobstore. When you deploy a release, BOSH orchestrates the compilation of packages and stores the result in the Blobstore.

Agent
BOSH includes an Agent on every VM that it deploys. The Agent listens for instructions from the Director and carries out those instructions. The Agent receives job specifications from the Director and uses them to assign a role, or Job, to the VM.
For example, to assign the job of running MySQL to a VM, the Director sends instructions to the Agent on the VM. These instructions include which packages to install and how to configure those packages. The Agent uses these instructions to install and configure MySQL on the VM.

Components used for cross-component communication
Message Bus (NATS)
The Director and the Agents communicate through a lightweight publish-subscribe messaging system called NATS. These messages have two purposes: to perform provisioning instructions on the VMs, and to inform the Health Monitor about changes in the health of monitored processes.
Registry
When the Director creates or updates a VM, it stores configuration information for the VM in the Registry so that it can be used during bootstrapping stage of the VM.

Example component interaction
This example shows how components interact when creating a new VM.
Creating a new VM

Through the CLI, the operator takes an action (e.g. deploy for the first time, scaling up deployment) which requires creating a new VM.
The CLI passes the instruction to the Director.
The Director uses the CPI to tell the IaaS to launch a VM.
The IaaS provides the Director with information (IP addresses and IDs) the Agent on the VM needs to configure the VM.
The Director updates the Registry with the configuration information for the VM.
The Agent running on the VM requests the configuration information for the VM from the Registry.
The Registry responds with the IP addresses and IDs.
The Agent uses the IP addresses and IDs to configure the VM.




Step 1: Setup a BOSH-Lite Vagrant VM
   Install BOSH

bosh-init is used for creating and updating a Director VM (and its persistent disk) in an environment
bosh-init for Linux (amd64)
Download the binary for your platform and place it on your PATH. For example on UNIX machines:
$ chmod +x ~/Downloads/bosh-init-*
$ sudo mv ~/Downloads/bosh-init-* /usr/local/bin/bosh-init


Check bosh-init version to make sure it is properly installed:
$ bosh-init -v


Depending on your platform install following packages:
Ubuntu Trusty
$ sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt-dev libxml2-dev libssl-dev libreadline6 libreadline6-dev libyaml-dev libsqlite3-dev sqlite3



Make sure Ruby 2+ is installed:
$ ruby -v
	ruby 2.2.3p173 (2015-08-18 revision 51636) [x86_64-darwin14]

Prepare the Environment
Install Vagrant
Known working version:
$ vagrant --version
Vagrant 1.7.4


Clone this repository
$ cd ~/workspace
$ git clone https://github.com/cloudfoundry/bosh-lite
$ cd bosh-lite


Install and Boot a Virtual Machine
Installation instructions for different Vagrant providers:
VirtualBox (below)
AWS
Using the VirtualBox Provider
Make sure your machine has at least 8GB RAM, and 100GB free disk space. Smaller configurations may work.
Install VirtualBox
Known working version:
$ VBoxManage --version
5.1...


Note: If you encounter problems with VirtualBox networking try installing Oracle VM VirtualBox Extension Pack as suggested by Issue 202. Alternatively make sure you are on VirtualBox 5.1+ since previous versions had a network connectivity bug.
Start Vagrant from the base directory of this repository, which contains the Vagrantfile. The most recent version of the BOSH Lite boxes will be downloaded by default from the Vagrant Cloud when you run vagrant up. If you have already downloaded an older version you will be warned that your version is out of date.
$ vagrant up --provider=virtualbox


When you are not using your VM we recommmend to Pause the VM from the VirtualBox UI (or use vagrant suspend), so that VM can be later simply resumed after your machine goes to sleep or gets rebooted. Otherwise, your VM will be halted by the OS and you will have to recreate previously deployed software.
Target the BOSH Director. When prompted to log in, use admin/admin.
# if behind a proxy, exclude both the VM's private IP and xip.io by setting no_proxy (xip.io is introduced later)
$ export no_proxy=xip.io,192.168.50.4

$ bosh target 192.168.50.4 lite
Target set to `Bosh Lite Director'
Your username: admin
Enter password: *****
Logged in as `admin'


Add a set of route entries to your local route table to enable direct Warden container access every time your networking gets reset (e.g. reboot or connect to a different network). Your sudo password may be required.
$ bin/add-route


Customizing the Local VM IP
The local VMs (virtualbox, vmware providers) will be accessible at 192.168.50.4. You can optionally change this IP, uncomment the private_network line in the appropriate provider and change the IP address.
 config.vm.provider :virtualbox do |v, override|
    # To use a different IP address for the bosh-lite director, uncomment this line:
    # override.vm.network :private_network, ip: '192.168.59.4', id: :local
  End


Step 2: Deploy Cloud Foundry
Create a Deployment Manifest for Cloud Foundry on BOSH-Lite
Target Your BOSH Director
Use the bosh target command with the address of your BOSH Director to connect to the BOSH Director. Log in with the default user name and password, admin and admin, or use the username and password that you set when you installed BOSH-Lite.
$ bosh target https://bosh.my-domain.example.com
Target set to `bosh'
Your username: admin
Enter password: *****
Logged in as 'admin'


Clone the cf-release GitHub Repository
$ git clone https://github.com/cloudfoundry/cf-release.git


Generate the Manifest
Ensure that you have the most up-to-date version of the Cloud Foundry code and all required submodules.
From the cf-release directory that you cloned when you created the manifest, run the update script to fetch all the submodules.
$ cd cf-release
$ ./scripts/update


Run gem install bundler to install bundler.
Install spiff.
Use the scripts/generate-bosh-lite-dev-manifest command to create a deployment manifest and set it as the current BOSH deployment.
$ cd cf-release
$ ./scripts/generate-bosh-lite-dev-manifest




Deploying Cloud Foundry
Upload a Stemcell
A stemcell is a versioned image of a bare-minimum OS skeleton, wrapped with IaaS-specific packaging. Deploying Cloud Foundry starts with specifying a stemcell, which BOSH installs on each component VM.
Open https://bosh.io/stemcells in a web browser to view the current list of publicly available BOSH stemcells.
Choose a BOSH stemcell for your IaaS and click the build number to download it to your computer. The stemcell downloads as a gzipped tar archive which you should not unzip.
In a terminal window, run bosh upload stemcell STEMCELL-PATH to upload your stemcell to the BOSH Director, where STEMCELL-PATH is the location of your downloaded stemcell file.
bosh upload stemcell ~/downloads/light-bosh-stemcell-3202-aws-xen-hvm-ubuntu-trusty-go_agent.tgz


Build the Cloud Foundry Release
Use bosh create release to create a Cloud Foundry release. After some processing, this command prompts you for a development release name. The release names correspond to the files in cf-release/releases.
Upload the Cloud Foundry Release
Use bosh upload release to upload the generated release to the BOSH Director.
$ bosh upload release


Deploy!
Use bosh deploy to deploy the uploaded Cloud Foundry release.
$ bosh deploy


Verify the Deployment
Run bosh vms. This command provides an overview of the virtual machines that BOSH manages as part of the current deployment. The state of every VM should show as running.
Use curl to test the API endpoint of your Cloud Foundry installation at api.YOUR-SYSTEM-DOMAIN/info.
$ curl api.INCORRECT-SYSTEM-DOMAIN/info
404 Not Found: Requested route ('api.INCORRECT-SYSTEM-DOMAIN.com') does not exist.

$ curl api.YOUR-SYSTEM-DOMAIN
{
 "code": 10000,
 "description": "Unknown request",
 "error_code": "CF-NotFound"
}

$ curl api.YOUR-SYSTEM-DOMAIN/info
{"name":"vcap","build":"2222","version":2,"description":"Cloud Foundry","authorization_endpoint":"https://login.YOUR-SYSTEM-DOMAIN","token_endpoint":"https://uaa.YOUR-SYSTEM-DOMAIN","allow_debug":true}


If curl succeeds, it should return specific JSON-formatted information about your deployment. If curl does not succeed, check your networking and make sure your domain has an NS record for your subdomain. 

If you do not know YOUR-SYSTEM-DOMAIN for your Cloud Foundry installation, examine the system_domain property in your deployment manifest.
You should be able to target your Cloud Foundry installation with the Cloud Foundry Command Line Interface (cf CLI) and log in as an administrator. The username is admin and the password is specified in the deployment manifest:
properties:
  ...
  uaa:
    ...
    scim:
      ...
      users:
      - admin|ADMIN_PASSWORD|...

Error During Installation
Problem : [WARNING] cannot access director,  trying 4 more times#143:
Solution :Use vagrant ssh and Run the command in this envrionment



     2.   ./scripts/generate-bosh-lite-dev-manifest NOT GENERATING     cf-release/bosh-lite/deployments/cf.yml' file
         Solution:run the bin/add-route script in the bosh-lite directory? That will route traffic destined to 10.244.0.34 through the BOSH-Lite vagrant VM.


3. Problem : Get stuck at 96% when running "bosh upload stemcell"
   Solution :1. Do bosh upload stemcell 
		2. While uploading and when getting stuck at 96% , open a new terminal and
login into vagrant vm with vagrant ssh & do monit restart for each director
worker ( 1 at a time ).

4. Problem:prepare deployment fails because of bosh-release gocd/docker symlink
Solution:bundle exec bosh prepare deployment


5.bosh upload release --fix doesn't upload jobs
bosh upload release dummy/0+dev.1
Switch blobstore in bosh manifest and re-deploy with bosh-init
bosh upload release dummy/0+dev.1 --fix











