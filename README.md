Role Name
=========

A utility role to help with OpenShift OpenShift Container Platform 4+ (OCP4) pre installation and supporting services.
Some of the initial playbooks are focused on deploying an agent based baremetal cluster and adding any required missing newtworking supporting cast (e.g. container registry, LB...). There are more options for this today than in years past but in some environments where things are not ready yet we need to be able to continue to deliver value to customers quicker by adding these stand in services. For instance the mirror-registry (Quay based registry supported by Red Hat) would work but does not work on NFS shares, which is a sutuation I have on an engagement I am currently working. That is partially what prompted my standing up of this repository and gradually add any necessary services I need to keep moving forward toward to targeted objective. The other reason is that we recently had to deploy bare metal OpenShift for a customer on short notice and very short timeline and due to the lack of clarity in documentation and working example of the various configurations required to successfully do an agent based deployment we had to pivot to another approach due to the timeline/schedule constraints as well as the fact that this was being done in a disconnected environment where it would have been far easier to have templatized manifests to help with this. So the agent-config.yaml nad associated install-config.yaml will be created here via a playbook to simplify the process.   
So this is a work in progress repository. 


Requirements
------------


Plays and Role Variables
------------------------

Variables for container registry configuration
- ansible_name_module: The name of the module this role is part of. This is used to track context when this role is run in a playbook as part of other plays.  
- staging_dir: The directory where rendered yaml config are placed on the '/tmp'
- local_repository: The name of repository where the operator images are stored in the disconnected registry (default to 'openshift4/redhat-operators')
- registry_host_fqdn: Host FQDN to use. It will also be used in the cert created and related SAN
- registry_host_port: Port to bind the registry to on the host. Default to 5000
- registry_container_image: The container image to use when launching a containerized registry. Default to docker.io/library/registry:2
- enable_authz_on_registry: Whether to use htpasswd authentication or not. Default to false.
- enable_tls_on_registry: Whether to use TLS  authentication or not. Default to false.
- registry_container_name: The name of the registry container when running containerized. Default to mirror-registry.
- registry_root_dir: The root directory where registry configuration files are stored. Default to /opt/registry for container and /etc/docker-distribution/registry for docker-distribution.
- registry_container_cert_dir: The directory where the registry certificates are stored within the container. Default to certs.
- registry_container_cert: The pem based registry certificate to use to support tls on the registry.
- registry_container_key: The pem based registry certificate key to use to support tls on the registry. 
- registry_data_dir: The directory where any registry related config data (e.g. config.yml) is stored relative to the root. Defaults to data.
- registry_container_storage_dir: The directory on the container where data is persisted . Defaults to /var/lib/registry.
- registry_auth_dir: The directory where the htpasswd is mounted into inside the container. Defaults to /auth.
- use_custom_ca_bundle: Whether to use custom CA bundle or not
- custom_ca_file: The pem based custom CA certificate bundle to use for client CA on the registry.
- load_container_image: Whether to load the registry image from an archive or not
- container_image_archive_file: The path to the registry container image archive on the controller. 
- registry_admin_username: The admin user to the registry to be used for the htpasswd authentication on the registry.
- registry_admin_password: The admin user password to be used for the htpasswd authentication on the registry.

Variables for agent based install configuration
- enable_fips:
- use_static_ips:
- staging_root_dir:
- deployment_staging_dir:
- cloudctl_staging_dir:
- platform_staging_dir:
- agent_install_dir:
- cluster_dir:
- cloud_provider:
- target_environment:
- domain_name:
- cluster_name:
- make_masters_schedulable:
- gateway:
- interface:
- netmask:
- dns_servers: []
- masters: The structure containing information about the various master nodes to add and configure. 
  - name: The name attribute of the master node.
  - mac: The mac address of the interface used to configure the node.
  - ip: The ip address of the interface used to configure the node.
- workers: The structure containing information about the various worker nodes to add and configure. 
  - name: The name attribute of the worker node.
  - mac: The mac address of the interface used to configure the node.
  - ip: The ip address of the interface used to configure the node.
- master_instance_number:
- worker_instance_number:
- api_vip_ip:
- ingress_vip_ip:
- use_bootstrap_osimage:
- bootstrap_osimage_uri:
- bootstrap_osimage_sha256:

Dependencies
------------
This role uses the https://github.com/cadjai/config-OpenShift-Lifecycle-Manager-Disconnected.git role to configure the disconnected OLM.


Installation and Usage
-----------------------
Clone the repository to where you want to run this from and make sure you can connect to your cluster using the CLI .
Ensure you install all requirements using `ansible-galaxy install -r requirements.yml --force` before performing the next steps.
You will also need to have cluster-admin run in order to run the playbook since the cron job need to run privileged.
Finally before running the playbook make sure to set and update the variables as appropriate to your use case.

Playbooks
---------
To run the main playbook use the ansible-playbook command as follows
`ansible-plabook install-and-configure-operators-from-olm.yml`

To run the infra nodes addition and configuraton  playbook use the ansible-playbook command as follows
`ansible-plabook add-and-configure-infranodes.yml`

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
