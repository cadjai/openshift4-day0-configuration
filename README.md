Role Name
=========

A utility role to help with OpenShift OpenShift Container Platform 4+ (OCP4) pre installation and supporting services.
Some of the initial playbooks are focused on deploying an agent based baremetal cluster and adding any required missing newtworking supporting cast (e.g. container registry, LB...). There are more options for this today than in years past but in some environments where things are not ready yet we need to be able to continue to deliver value to customers quicker by adding these stand in services. For instance the mirror-registry (Quay based registry supported by Red Hat) would work but does not work on NFS shares, which is a sutuation I have on an engagement I am currently working. That is partially what prompted my standing up of this repository and gradually add any necessary services I need to keep moving forward toward to targeted objective. The other reason is that we recently had to deploy bare metal OpenShift for a customer on short notice and very short timeline and due to the lack of clarity in documentation and working example of the various configurations required to successfully do an agent based deployment we had to pivot to another approach due to the timeline/schedule constraints as well as the fact that this was being done in a disconnected environment where it would have been far easier to have templatized manifests to help with this. So the agent-config.yaml nad associated install-config.yaml will be created here via a playbook to simplify the process.   
So this is a work in progress repository. 

#### Important: Agent Installer configuration notes

If using the agent installer to configure and deploy baremetal in disconnected environments ensure the following are met:
1. For the agent installer approach to work in a disconnected environment it is important to ensure that the openshift installer used (openshift-install) has the target/destination registry URL/FQDN embedded in it so that you don't get the `unable to read image quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:` error. 
If you need to extract the installer for a disconnected environment refer to the [extract-installer-for-disconnected-environment.yml](https://github.com/cadjai/mirror-openshift-container-platform-operators) playbook for how to properly mirror the OCP tools. 
Note that you need to be on a connected host for the tools extraction to work. The easiest way to do this until the bug with the extract sub command of the oc client is fixed is to use a mirror-registry running on localhost in the disconnected environment and to use localhost as the FQDN for the target/destination registry when running the extract sub command.  
However, if you know the FQDN of the target registry you can still simulate that by dowloading the openshift payload unto the connected host and then running a container registry or mirror-registry with that target FQDN and then running the extraction command `oc adm release extract --tools` to extract the tools (or at least the installer) to be transferred to the disconnected environment.  
Also note that if you are running the container registry (or the mirror-registry) using the target registry FQDN you can add the FQDN to your /etc/hosts so that the FQDN is resolved to your connected hosted being used to extract the tools. Just add the target registry FQDN to the end of the first line in your /etc/hosts to look like below (e.g. target registry FQDN is my.registry.disco.net)
`127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4 my.registry.disco.net`

2. The nmstatectl should have a  version of at 1.4.2-4 on RHEL8 and 2.2.7 on RHEL9 bastion. If not you get some errors during the agent iso creation. Some of the errors look like errors discussed in [this](https://access.redhat.com/solutions/7020319) and [this](https://access.redhat.com/solutions/7012255) knowlegde articles even though you might have already fixed the issues in the articles. 

Note that is you are using the agent installer and can't find the correct version of the nmstatectl package it is better to either bring it in or use the UPI approach even if this is more manual and slower than the agent based installer. 
If on the other hand you can afford to go collect the missing package you can do so using the following commands and then copy the resulting archive to a media and transport it to the disconnected environment.
```
mkdir /tmp/nmstatctl
sudo yumdownloader --alldeps --resolve --destdir /tmp/nmstatctl nmstate-1.4.5-2.el8_9.x86_64
tar czvf /tmp/nmstatectl-rpm-with-dependencies.tar.gz --directory=/tmp/nmstatctl .
```

Note that you could have used nmstate instead of nmstate-1.4.5-2.el8_9.x86_64 and got almost the same result.

 3. Ensure that NTP is configured for each host including bastion using both the BIOS setting to configure time as well as the ipmi interface to set the NTP servers fo the hosts.

 4. Ensure that the hosts are all using the same fireware version if possible because a different mean a lack of feature needed during the configuration. Also ensure that firmware version meets minmun requirements for redfish and/or ipmi as specified in [the firmware section](https://docs.openshift.com/container-platform/4.12/installing/installing_bare_metal_ipi/ipi-install-prerequisites.html#ipi-install-firmware-requirements-for-installing-with-virtual-media_ipi-install-prerequisites).


#### Important: Baremetal UPI configuration notes

If using the baremetal UPI approach in disconnected environments, ensure the following are met.
1. Ensure that the disk to use on each of the host is clean. For instance if the hosts have previously been used, chances are that there will be partitions on the disk, which will result in I/O errors during the coreos-installer instal command run. Ensure that you first delete any existing partition using utilities like `fdisk` or `parted` with a live RHEL CD/DVD to remove any preexisting partitions on the disk or use `dd if=/dev/zero of=/dev/sda bs=1M count=4096 to overwrite existing partition table and data. When there are partitions you might see an error with `device busy` or something similar. 

2. Ensure that NTP is configured for each host including bastion using both the BIOS setting to configure time as well as the ipmi interface to set the NTP servers fo the hosts.

3. Ensure that the hosts are all using the same fireware version if possible because a different mean a lack of feature needed during the configuration.

4. Use /dev/disk/by-path or /dev/disk/by-id to provide the target installation disk to the coreos-installer install command. Otherwise you might get some random I/O errors as well, which is usually caused by the fact that the disk order is not what you expect it to be. Therefore passing `/dev/sda` for instance causes the installer to not find that disk and throw the I/O error. Here is a snippet of the error message `blk_update_request: I/O error, dev loop1, sector`.

5. If possible configure DHCP (and DHCP relay) for the subnet used for the deployment to make life easier. If that is the case follow the official [dhcp guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/networking_guide/sec-dhcp-configuring-server#config-file) and the [official DHCP Relay Config guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/managing_networking_infrastructure_services/providing-dhcp-services_networking-infrastructure-services#setting-up-a-dhcp-relay-agent_providing-dhcp-services).

6. Also if unable to switch from using virtual media to http boot while using Dell IDRAC it is better to do a PXE boot instead as the virtual media speed will not allow the CoreOS to fully boot and you get various errors that will prevent the cluster from coming up. For that reason if setting up PXE for RHCOS use the [RHEL 8 PXE boot official guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/automatically_installing_rhel/preparing-for-a-network-install_rhel-installer#configuring-the-dhcpv4-server-for-http-and-pxe-boot_preparing-for-a-network-install) and modify steps as follows.
   1. Configure DHCP and TFTP per instructions above.
   2. Obtain the syslinux.efi from a RHEL 8 DVD ISO to be used to bootstrap the PXE process for the nodes. This file needs to be placed at the root of the tftpboot.
   3. Follow official steps listed in [RHCOS PXE guide](https://docs.redhat.com/en/documentation/openshift_container_platform/4.6/html/installing/installing-on-bare-metal#installation-user-infra-machines-pxe_installing-bare-metal) to obtain the necessary RHCOS PXE 
       assets to be hosted on the tftp and/or web server.
   4. Create grub files for each of the node type (bootstrap, master, worker) to be placed under the pxelinux.cfg folder with a content similar to the snippet below
      
   ```
LABEL rhcos
  KERNEL <path/to/vmlinuz file>
  IPAPPEND 3
  APPEND initrd=<path/to/initrd.img file> initrd=<path/to/ignition.img file> coreos.live.rootfs_url=<url of rootfs.img file in http> ignition.firstboot ignition.platform.id=metal console=tty0 random.trust_cpu=on rd.luks.options=discard coreos.inst.install_dev=/dev/disk/by-path/<disk-id> coreos.inst.ignition_url=<url of ignition file (bootstrap.ign,master.ign,worker.ign)> ip=<interface device name>:dhcp nameserver=<comma separated nameserver list>
  MENU LABEL RHCOS    - RHCOS node boot
LABEL auto
  KERNEL <path/to/vmlinuz file>
  IPAPPEND 2
  APPEND initrd=<path/to/initrd.img file> initrd=<path/to/ignition.img file> coreos.live.rootfs_url=<url of rootfs.img file in http> ignition.firstboot ignition.platform.id=metal console=tty0 random.trust_cpu=on rd.luks.options=discard coreos.inst.install_dev=/dev/disk/by-path/<disk-id> coreos.inst.ignition_url=<url of ignition file (bootstrap.ign,master.ign,worker.ign)> ip=<interface device name>:dhcp nameserver=<comma separated nameserver list>
  MENU LABEL ^AUTO     - Normal node boot
  MENU DEFAULT
DEFAULT auto
PROMPT <use 30 for bootstrap, 1 for master and 0 for worker>
TIMEOUT <use 90 for bootstrap, 50 for master and 50 for worker>
```
   


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
- overwrite_cluster_osimage:
- cluster_osimage_uri:
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
