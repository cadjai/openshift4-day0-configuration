**VPC DEPLOYMENT**

VPC must have the tag: kubernetes.io/cluster/{{ cluster_name }} This includes the VPC, subnets, and other major resources.  Ensure that the cluster name is correct on the tags.


Iam roles and policies, Security Groups, instance profiles.

1. IAM policy

Create profiles for the masters and worker machines in IAM > policies. Copy an existing profile in the account/environment, as they are account/VPC agnostic, and name it {cluster_name}-{node-role}-policy.

2. IAM Role

Create roles for the masters and worker machines in IAM > Roles. Name the role {cluster_name}-{node-role}-role, and attach the previously created policy.

3. IAM Profile

Ensure via aws CLI that the IAM Instance profiles exist, and are named $(environment)-master-role and $(environment)-worker-role by default.  The automation looks for profiles named $(environment)-master/worker-role.  This will need a bastion with aws configure run with the existing AWS CLI credentials.

# Create new properly named profiles
aws iam create-instance-profile --instance-profile-name dev-master-role
aws iam create-instance-profile --instance-profile-name dev-worker-role

# Add new profiles to roles
aws iam add-role-to-instance-profile --instance-profile-name dev-master-role --role-name dev-master-role
aws iam add-role-to-instance-profile --instance-profile-name dev-worker-role --role-name dev-worker-role

**BASTION DEPLOYMENT and CONFIGURATION**

In order to confiugre bastions with session manager access, they must have:

The instance must have the ssm-agent package installed.
An instance role which has the policy: AmazonSSMManagedInstanceCore.  The existing role in NIPR is ssm-agent-role.  When creating a bastion, use the latest spel-minimal-rhel-8-hvm AMI.

### NOTE Make sure to add an inbound security group rule to your bastion (or attach the registry-sg to the bastion) to allow port 8080/TCP so the control plane nodes can retrieve the ignition files from the bastion. Also ensure that your NIPR workstart can reach the bastion via SSH port 22/TCP

1. Provision a bastion in the VPC you are deploying into and install the necessary packages. Some of the necessary packages includes ansible, git, vim, jq, unzip, awscli ...You might need to configure the appripriate repositories to install all of those packages.

Bastion Info:
```
AMI Name: spel-minimal-rhel-(el-version)-hvm-(year).(month).(day).(arch)-(storage) where el-version is the rhel version used . The minimium is 8 and for OpenShift4.19+ use rhel9 or rhel10.
Size of Volume: 500 GiB gp3
Security Group Rules
   - Ports 8080/tcp, 22/tcp, and 3389/tcp need to be open to local vpc traffic 
```
After installing epel-release, you can then proceed to install the additional needed packages:

```bash
sudo yum -y install \
ansible \
git \
vim \
jq \
skopeo \
unzip \
setools \
bind-utils \
podman \
openscap-scanner \
tmux \
httpd-tools \
wget
```

If using the SPEL AMI (default bastion AMI for most environments), you will likely need to extend the /home, /var, and / partitions as they come very small by default.  This can be accomplished with the following commands (verify the partition name and sizes before running):

```bash
sudo growpart /dev/xvda 2
sudo pvresize -v /dev/xvda2
sudo lvextend -L +15G -r /dev/VolGroup00/homeVol
sudo lvextend -L +15G -r /dev/VolGroup00/varVol
sudo lvextend -l +100%FREE -r /dev/VolGroup00/rootVol
```

```bash
sudo growpart /dev/nvme0n1 2
sudo pvresize -v /dev/nvme0n1p2
sudo lvextend -L +15G -r /dev/RootVG/homeVol
sudo lvextend -L +15G -r /dev/RootVG/varVol
sudo lvextend -l +100%FREE -r /dev/RootVG/rootVol
```

Also, the redhat-admin needs to be created and given sudo privileges:
```bash
sudo useradd -m redhat-admin
passwd redhat-admin # Use Default Passwd
visudo # Command to edit sudoers file
```
Add the following to the sudoers file:
```
redhat-admin   ALL=(ALL)   NOPASSWD: ALL
```
And then type ```:wq``` to exist the file and apply the changes

**NOTE** On RHEL9+ (or any RHEL8.7+) if newer FIPS crypto libraries are being used you will need to run the following command in order to enable legacy crypto, otherwise you won't be able to clone or pull from gitlab or any other enterpise resources still using TLSv2. 
```bash
update-crypto-policies --set FIPS:NO-ENFORCE-EMS
```

**Note** If the step above is skipped you will see an error similar to ` error:1C8000E9:Provider routines::ems not enabled` when using git clone or even curl against a resource not enforcing EMS

**Note** If not using gitops where the variables and configurations are not stored in a separate repository skip steps 2 and 3 below and go to step 4 below 

2. Clone the cac repository for an existing VPC in the same account. Use this as a template to assist you in building out your variables if one has not already been created for your VPC.

``` 
  git clone https://gitlab. cluster-deployment-${cluster_name}-${today-date} 
```

3. Change directory into the environment directory for the cluster being deployed of the cloned repository and create a directory for the date the cluter is being deployed. Copy the variables files from the aide/defaults_for_new_deployment/ into the the vars folder of the created date directory and then update the variables to reflect the intended cluster. The files to update are usually the vault.yml, cluster.yml and perhaps some variables in the fabric.yml (refer to the instructions in the cluster deployment repository for more information). 
Once the variables are updated, commit and push your changes to the branch created for the cluster. 
Several variables are required while other have default values. Some of the environment specific variables are left empty to force you to set the values. 

4. Clone the ocp4-day0-configuration repository on to bastion provisioned and configured in step 1 above.
* See Appendix regarding bastion prerequsites and RPM installation

``` 
   git clone https://github.com/cadjai/openshift4-day0-configuration 
```

Note that the git_repo value of cluster-var-repo-url-with-token is only needed if you are doing this on a bastion. If running this on AAP you won't need to provide that value. 

5. If not using automation for DNS and ELB, you need to create and submit appropriate requests for DNS records for the LoadBalancers as well. For instance if ELB cannot be automated but you are still allowed to use it , you will need to provision the various ELBs and associated Tareget Groups and submit DNS record entries for the ELBs and then once the deployment is started, as the nodes come up you will have to update the various target groups appropriately. More details will be added later. 

**CLUSTER DEPLOYMENT PREP**  

1. Install the necessary client using the install-clients.yml.
```bash
ansible-playbook -i inventory -vv --ask-vault-pass install-clients.yml
```
2. Extract and overwrite the necessary openshit clients. When running in disconnected environments the client to use (oc and openshift-install) need to be extracted from the openshift payload hosted in the private disconnected registry in order for them to behave as expected in those environments. In order to do that we need to overwrite the clients installed in the previous step using the following command.
```bash
ansible-playbook -i inventory -vv --ask-vault-pass install-disconnected-openshift-clients.yml 
```
3. Install and configure ansible collection on the host to that collections can be used durng the playbook runs. There are two ways to install collections . If AAP is available then use the first command below which will pull the collections from AP but in case AAP is unavailable or is having issue use tje second command which downloads a bundle from the artifact repository (e.g. artifactory), then extract the collections and uses that ti install them on host in an offline manner. 
To install collections from the AAP host use the following command 
```bash
ansible-playbook -vvv --ask-vault-pass  -e cluster_dir=<cluster-dir-name-in-cac-repo> -e git_repo=<your-git-url-with-token> install-ansible-collections.yml
```

To install collections from the artifact repository host use the following command 
```bash
ansible-playbook -vvv --ask-vault-pass  -e cluster_dir=<cluster-dir-name-in-cac-repo> -e git_repo=<your-git-url-with-token> download-and-install-local-collections.yml 
```
4. Install and configure terraform aws provider locally on the bastion so that the local provider can be made available to the konductor container during deployment. This step is necessary for disconnected deployment where a preconfigured terraform provider binary might not be available.  
Note that this step assumes that the necessary terraform provider binary is available on the bastion and the appropriate variables are set. Use the config-local-aws-terraform-provider.yml playbook to configure the terraform aws provider.
```bash
ansible-playbook -i inventory -vv --ask-vault-pass config-local-aws-terraform-provider.yml 
```

5. Run the webserver container playbook to enable the nginx webserver to be used on the bastion to host the ignition configuration for the cluster to be deployed from the bastion. To run the webserver container run the following command. 
```bash
ansible-playbook -i inventory -vv --ask-vault-pass run-nginx-container.yml -e platform_container_staging_dir=/root/platform-<cluster-name>/container-config -e is_local_vol=true
```

6. Run the setup-aws-upi.yml playbook from the bastion to configure the deployment directories and any applicable overlays.
```bash
ansible-playbook -vvv --ask-vault-pass  -e cluster_dir=<cluster-dir-name-in-cac-repo> -e git_repo=<your-git-url-with-token> setup-aws-upi.yml 
```
##### Note: The above playbook can now be run as a regular user and the staging directory can have a root under the user home dir (e.g. /home/redhat-admin/platform-<cluster-name>) which enables running the playbook as well as the cluster deployment playbook as non root later. Ensure the value of `staging_root_dir` var is set to the ansible user local home or whatever is under that user home for permissions handling.  

**NLB Creation and DNS Requests (Non-Route53 Deployment)**

1. Create 4 new target groups with listeners to their respective ports (all TCP protocol). 2 will be for the API NLB and 2 will be for the ingress NLB that you will be creating
   - $(cluster_name)-80-ingress-tg
   - $(cluster_name)-443-ingress-tg
   - $(cluster_name)-6443-int-tg
   - $(cluster_name)-22623-int-tg

2. Create 2 new NLBs
   - $(cluster_name)-ingress
   - $(cluster_name)-int

3. Once the NLBs are created and forwarding to their proper target groups, use the NLB's DNS name to submit the following(via ticket or to Dawne Shinno):
   - api-int.$(cluster_name).$(cluster_base_domain)   CNAME    $(DNS name for int NLB)
   - api.$(cluster_name).$(cluster_base_domain)       CNAME    $(DNS name for int NLB)
   - *.apps.$(cluster_name).$(cluster_base_domain)    CNAME    $(DNS name for igress NLB)
   - registry.$(cluster_name).$(cluster_base_domain)  A Record $(Bastion IP Address)

**CLUSTER DEPLOYMENT**

1. Copy the generated cluster-vars file into place 
   ```bash
   /home/redhat-admin/platform-<cluster-name>/iac/cluster-vars.yml && cp -f /home/redhat-admin/platform-<cluster-name>/cluster-vars.yml /home/redhat-admin/platform-<cluster-name>/iac/
   ```

2. Initialize terraform for the deployment
   ```bash
   for d in $(ls -l /home/redhat-admin/platform-<cluster-name>/iac/tf-upi/ | egrep '^d' | egrep -v "templates|vars" | awk '{print $9}'); do cd /home/redhat-admin/platform-<cluster-name>/iac/tf-upi//$d && <path-to-terraform-binary> init --plugin-dir=<path-to-terraform-plugin>/.terraform.d/plugins ; done
   ```
3. Run the cluster provisioning automation playbook
   ```bash
   cd /home/redhat-admin/platform-<cluster-name>/iac/openshift && ./site.yml -vv
   ```

