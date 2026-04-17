# AWS STS OIDC Configuration Implementation steps (within secure AWS private government regions) 
For more information on the solution refer to the [STS solution readme](STS-design-solution.md)

To deploy an OpenShift cluster with STS enabled in a secure AWS government region, we must first solve the challenge of exposing OIDC documents from a private S3 bucket. Our selected solution, detailed in the architecture documentation, is to use an AWS API Gateway, as it is consistently available across all our target environments. Below is a depiction of the flow with only 1. being required but 2. is recommended for validation and testing.

1.  OpenShift Pod -> AWS STS -> Regional API Gateway -> Private S3 Bucket  (required)

2.  OpenShift Pod -> AWS STS -> Private API Gateway -> Private S3 Bucket   (optional)

The deployment process is heavily automated through our Infrastructure as Code (IaC). It uses the standard OpenShift ccoctl tool to generate the initial AWS resources, which our automation then customizes to integrate the API Gateway and applied to the VPC using the AWS CLI. This pre-deployment stage configures all necessary STS components, ensuring that the required manifests are available to the cluster installer.

The workflow involves updating configuration variables in our CaC (Configuration as Code) repository, which are then used by the deployment playbooks to create the STS-enabled cluster.

Most of the steps below are done before running the [`setup-aws-upi.yml`](setup-aws-upi.yml) playbook so that during the cluster deployment step the various STS manifests are created and used during the custer deployment. 

## Step One: VPC Configuration Readiness Validation 
The appropriate VPC resources necessary for the STS permission must already be in place. If not ensure that you create them or submit a ticket to have them created.  
To help with the various policy document generation, a helper playbook is added [`render-sts-iam-policy-docs.yml`](render-sts-iam-policy-docs.yml) that will render all of the various IAM policy and role documents that can then be used to configure the appropriate account using the AWS CLI or the AWS console. In case this needs to be done by a different group,  a ticket can be submitted with the rendered policy documents to make the configuration easier.   
**NOTE**
:warning: Similar to all playbooks, the vault and other variables need to be updated to reflect the target environment before running the playbook .

1. IAM role for the IaC deployer ( to be assumed by the ec2 provisioner instance)
Create an IAM role that will be assumed during the cluster provisioning IaC to deploy the cluster with STS. Create a role with a name like `openshift-deployer-role` with a content similar to the snippet provided in the [STS IAM documents](STS-iam-documents.md) .  
**NOTE**
:warning: Don't forget to add the trust-relationship that includes the openshift role, the deployer role and the ssm role.

2. IAM policies for the cluster provisioner
Create the required IAM policies to support STS permissions to be used during the cluster deployment as well as for the various cluster components needing STS. The following policies will be needed.
a. Create an sts iam policy json with snippet provided in the [STS IAM documents](STS-iam-documents.md) to enable the IaC ec2 instance to assume the role to provision and configure STS OIDC provider IAM resources and related IAM roles. 
b. Create an apigateway iam policy json with snippet provided in the [STS IAM documents](STS-iam-documents.md) to enable the IaC ec2 instance to assume the role to provision and configure an apigateway used to make the content of the private s3 bucket publicly accessible via https. 
c. Create an s3 iam policy json with snippet provided in the [STS IAM documents](STS-iam-documents.md) to enable the IaC ec2 instance to assume the role to provision and configure a private s3 bucket used to store the OIDC provider JWT documents . 
d. Create/update existing an openshift iam policy json with snippet provided in the [STS IAM documents](STS-iam-documents.md) to enable the IaC ec2 instance to assume the role to provision and configure an opensift cluster . Note that this policy should already exist and might only need to be updated if necessary. 
e. Create/update existing an SSM iam policy json with snippet provided in the [STS IAM documents](STS-iam-documents.md) to enable the IaC ec2 instance to assume the SSM role to provision and configure an opensift cluster . Note that this policy might  already exist . 

3. VPC Endpoint for API gateway 
  - Ensure an interface vpc endpoint is created for the api gateway endpoint with service name `com.amazonaws.{{ aws_region }}.apigateway`. Without that you might not be able to use the API gateway endpoint through the AWS CLI to perform actions.
  - Ensure an interface vpc endpoint is created for the api gateway execute-api endpoing with service name `com.amazonaws.{{ aws_region }}.execute-api` . This is not required for the succesful provisioning, configuration and deployment of the api gateway but it is recommended because you need to make sure that the api gateway is correctly configured and that is where this endpoint becomes important since a mirror private api gateway is created as a mirror image of the regional one being used so that you can invoke the api and ensure that things are being returned as expected. 

**NOTE**
:warning: The private api gateway (created for testing) needs to be created with an endpoint-configuration set to `PRIVATE` (the IaC code takes care of that via a variable listed below). Subsequently the VPCE for the execute-api needs to have the `use private DNS` flag set to `yes`. Otherwise you will not be able to resolve or invoke the api from within the VPC. 

4. IAM roles, policies and trust-relationships for OpenShift Components needing STS
Create a IAM role, policy statement and trust-relationship for each of the OpenShift component that requires STS to create an AWS service using the snippet provided in the [STS IAM documents](STS-iam-documents.md). By default the following OpenShift components are configured during installation to use STS but for each addtional component (e.g. EFS) a similar policy statement is needed. 
- cloud-credential-operator
- cloud-network-config-controller
- cluster-csi-drivers-ebs
- image-registry-installer 
- ingress-operator
- machine-api  

**NOTE**
:warning: for each of the default OpenShift components listed above, if the openshift deploy IAM role created above is granted the permission to create IAM policies then those default role and related policies and trust relationship objects will be created for you during the cluster deployment by the IaC using a combination of ccoctl and AWS CLI. However, in the event that the deployer IAM role is not granted the permissions to create those role you will need to locate the ones staged by the ccoctl tool under the cluster deployment staging directory (`platform_staging_dir/cluster/ccoctl-manifests/`) with the name prefix pattern of 05-# and 06-# and provide those with the ticket to have them created.  
For additonal/optional components not deployed with OpenShift (e.g. EFS or any operator that need to consume AWS API resource that requires STS), follow the documentation to create the required IAM role, policy and trust relationship resources.  

**NOTE**
:warning: for each of the component IAM created above the arn is needed to configure the STS credential secret for the component during installation, which the installation IaC takes care of. 
Once the arn of the IAM role for the component is retrieved, it is used to configure the STS secret to inject into the compenent's pod in order for the component to be able to use STS. This is done automatically for cluster components listed above but needs to be done for any additonal component that requires STS.  


## Step Two: Cluster deployment preparation step 
During the cluster predeplpyment preparation stage, ensure that the following variables are configured if STS is needed for the cluster deployment.

1. Edit the cluster variables to enable STS.
  - set `use_sts_creds` to `true`  to enable STS
  - ensure the `ocp_iam_role_arn` is set the the appropriate deployer iam role arn in the vault
  - ensure `aws_apigateway_execute_vpce` is correctly set the the api gateway execute-api vpce if using vpce with the resource policy. This is optional and not required.
  - ensure `use_private_s3_bucket` to `true` given that public s3 buckets are not allowed.
  - set `create_private_s3_bucket` to `true` if you want the private s3 bucket to be created and configured. This is usually the case if this the first time this cluster is being deployed.
  - set `push_to_private_s3_bucket` to `true` if you want the oidc JWT documents ito be pushed to the private s3 bucket. This is usually the case if this the first time this cluster is being deployed or if the component IAM policies are being recreated.
  - ensure `use_dry_run` is set to `true`.
  - set `create_aws_openid_idp` to `true` if you want the openid provider aws resource to be created. Note that if there is one already the iac will fail since this is not allowed. The only fileds that can be updated post creation are the fingerprint, the tags or the clients.
  - set `create_ccoctl_iam` to `true` if you want the components IAM polices to be created. If the iac is not allowed to create these then this have to be precreated and then this value set to false so the iac can retrieve the arn or that can be passed in as variables.
  - set `configure_apigateway` to `true` if you want to create and configure an api gateway. Note that if there was one previously created and this is set to true the one is deleted and a new one created. Howevers the consequence is that the oidc provider will need to be recreated since it is tied to the url of the apigateway.
  - ensure `use_ccoctl` is set to `true`.  
  - ensure `aws_deploy_stage` is set to a correct value (usually prod but dev can also be used).
  - ensure `aws_partition` is set the the correct partition for the fabric you are deploying to. 
  - ensure `apg_ep_config_type` is set to `PRIVATE`. If set to `REGIONAL` it is treated as public and there DNS resolution issues when trying to invoke the api from within the VPC.

2. Commit your updated variables to the repository and follow continue with the cluster deployment steps .

3. Proceed to the actual deployment step  .

**Note**
During the `setup-aws-upi.yml` playbook run ensure that you are setting `run_overlay_var` to `true` to ensure you ar using the updated variables or rerun the `ocp-vars-overlay.yml` playbook to ensure the variables in the cluster deployment repository have been updated to reflect the STS configuration changes before running the `setup-aws-upi.yml` playbook.  
**NOTE**
:warning: if the current bastion was previously used to deploy a cluster without STS it is important to rerun `setup-aws-upi.yml` so that the aws credentials used by the ec2 instance will be configured to assume the correct role instead of using the default profile credentials previously configured, which will cause the cluster to fail to deploy due to invalid permissions. 
