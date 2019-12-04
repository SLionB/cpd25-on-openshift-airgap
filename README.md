# Deploying OpenShift 3.11 on AirGap / Disconnected environment

<b>(Reference:  https://docs.openshift.com/container-platform/3.11/install/disconnected_install.html )</b>

![alt text][logo]

[logo]: https://github.com/ekambaraml/openshift311-airgap/blob/master/VPN-Bastion.png "Deployment Architecture"

# Steps Required to deploy
* [ ] 1. Provision Infrastructure
* [ ] 2. Download the OpenShift RPMs and Docker images
* [ ] 3. Setup Bastion Host
* [ ] 4. Setup local YUM RPM Repository for OpenShift 3.11
* [ ] 5. Setup local Docker Registry
* [ ] 6. Load OpenShift images
* [ ] 7. NFS server setuo
* [ ] 8. Prepare cluster nodes
* [ ] 9. Load Balancer setup
* [ ] 10. Create OpenShift inventory file
* [ ] 11. Install OpenShift 
* [ ] 12. Validate deployment


# 1. Provision Infrastructure


| Host |Count |Configuration | Storage | Purpose |
|:------|------|:-------------|:------------|:----------|
| Bastion| 1 | <ul><li>8 Core</li><li>16 GB Ram</li><li>RHEL 7.5+</li></ul>|<ul><li> /ibm 100 GB install</li><li>/var/lib/docker 200+ GB to storge docker images </li><li>/repository 500+ GB for local repo/registry</li></ul>|Virtual Machine for installation; This machine will also act as a local RPM Yum repository and Docker registry for airgap environment|
| Master nodes|1 or 3 |<ul><li>8 Core</li><li>32 GB Ram</li><li>RHEL 7.5+</li></u>|<ul><li> 100 GB Root</li><li>/var/lib/docker 200+ GB for Docker storage </li></ul>|Virtual Machine running OpenShift Control plane|
| Worker nodes|3 or more| <ul><li>16 Core</li><li>64GB Ram</li><li>RHEL 7.5+</li></u>|<ul><li> 100 GB Root</li><li>/var/lib/docker 200+ GB for Docker storage</li><li>1 TB GB for persistance storage</li></ul>|Virtual Machine running Workload|
| Load Balancer|1 option| <ul><li>4 Core</li><li>8GB Ram</li><li>RHEL 7.5+</li></ul>|<ul><li> 100 GB Root</li></ul>|Virtual Machine local load balancer|

* [ ] 2. Setup External DNS 
       
       https://docs.openshift.com/container-platform/3.11/install/prerequisites.html#prereq-dns
       
* [ ] 3. Setup DNS wild card 
       https://docs.openshift.com/container-platform/3.11/install/prerequisites.html#wildcard-dns-prereq
       
* [ ] 4. Provision LoadBalancer (external if any)


# 2. Download Openshift Product
Openshift install requires RPM and access the redhat docker registry (registry.redhat.io). In the case of AirGap deployment, both need to be made available local to the environment.

# 3. Bastion Host setup
Ansible playbooks are created to automate the steps required to prepare the nodes and installing prerequisite packages and configuring the machines. The playbooks are stored in an external git repository for everyone to make use of it.  

Bastion node is a host in the same network as cluster nodes and have access to cluster nodes. This host will be used for installing OpenShift cluster and hosting the temporary local Yum repository and Docker registry for installation purpose.

* [ ] 1. Clone the git repository

```
	cd /ibm
	git clone https://github.com/ekambaraml/openshift311-airgap.git
	cd openshift311-airgap
```
	
* [ ]  2. Hostfile Creation

The playbooks requires hostfile listing machines. Here is an example hosts file format:

* Example: using SSH

```
	[cluster]
	172.300.159.178 private_ip=10.188.201.34 name=master-01 type=master 
	172.300.159.190 private_ip=10.188.201.39 name=master-02 type=master 
	172.300.159.184 private_ip=10.188.201.44 name=master-03 type=master 
	172.300.104.56  private_ip=10.188.201.6  name=worker-01 type=worker 
	172.300.104.48  private_ip=10.188.201.51 name=worker-02 type=worker
	172.300.104.46  private_ip=10.188.201.58 name=worker-03 type=worker
	[loadbalancer]
	172.300.104.59 private_ip=10.188.201.11 name=loadbalancer type=proxy 
```

* Example: using password

```
	[cluster]
	172.300.159.178 private_ip=10.188.201.34 name=master-01 type=master ansible_ssh_user=root ansible_ssh_pass=<password>
	172.300.159.190 private_ip=10.188.201.39 name=master-02 type=master ansible_ssh_user=root ansible_ssh_pass=<password>
	172.300.159.184 private_ip=10.188.201.44 name=master-03 type=master ansible_ssh_user=root ansible_ssh_pass=<password>
	172.300.104.56  private_ip=10.188.201.6  name=worker-01 type=worker ansible_ssh_user=root ansible_ssh_pass=<password>
	172.300.104.48  private_ip=10.188.201.51 name=worker-02 type=worker ansible_ssh_user=root ansible_ssh_pass=<password>
	172.300.104.46  private_ip=10.188.201.58 name=worker-03 type=worker ansible_ssh_user=root ansible_ssh_pass=<password>
	[loadbalancer]
	172.300.104.59 private_ip=10.188.201.11 name=loadbalancer type=proxy ansible_ssh_user=root ansible_ssh_pass=<password>
```

* [ ] 3. Update the vars/global.yml with your environment specific values
```
	---

	sudo: yes
	docker_disk: /dev/xvdc
	docker_storage: /var/lib/docker
	nfs_disk: /dev/xvde
	time_server: <your timeserver>
	yum_repository_server: <yum repository server - Bastion Host>

	#OSE YUM Repository
	yum_ose_dest: /etc/yum.repos.d/ose.repo
	yum_ose_src: ../utils/ose.repo"


	##RedHat Subscription
	redhat_username: <username>
	redhat_password: <password>
	redhat_pool_ids: <pool-id>
```


# 4. Setup local Yum repository for OpenShift 3.11 RPMs
  * [ ] Mount the disk to <b> /repository </b>. It requires about 200GB of storage
```
     $ cd /repository
     $ mkdir ocp311
     # <Transfer the downloaded openShift  RPMs under the /repository/ocp311 folder and untar the RPMs>
 ```
  * [ ] Install httpd for hosting the RPM repository
```
    yum install -y httpd
    cd /var/www/html
    ln -s /repository/ocp311 /var/www/html/repo
```
  * [ ] Ensure HTTP Server configuration file enables symlink with this option: `Options Indexes FollowSymLinks` for given directory
```   
    vi /etc/httpd/conf/httpd.conf
    yum install -y policycoreutils-python
    semanage fcontext -a -t httpd_sys_content_t "/repository/ocp311/ppa(/.*)?"
    restorecon -Rv /repository/ocp311/ppa
```
  * [ ] Open Firewall port

```
    systemctl unmask firewalld
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --zone=public --add-port=80/tcp --permanent
    firewall-cmd --reload
    systemctl enable httpd
    systemctl start httpd
```
  * [ ] Create /etc/yum.repos.d/ose.repo using the blow content. Note: Repo-server = Bastion node IP
```
    [rhel-7-server-rpms]
    name=rhel-7-server-rpms
    baseurl=http://<repo-server>/repo/rhel-7-server-rpms
    enabled=1
    gpgcheck=0

    [rhel-7-server-extras-rpms]
    name=rhel-7-server-extras-rpms
    baseurl=http://<repo-server>/repo/rhel-7-server-extras-rpms
    enabled=1
    gpgcheck=0

    [rhel-7-server-ansible-2.6-rpms]
    name=rhel-7-server-ansible-2.6-rpms
    baseurl=http://<repo-server>/repo/rhel-7-server-ansible-2.6-rpms
    enabled=1
    gpgcheck=0

    [rhel-7-server-ose-3.11-rpms]
    name=rhel-7-server-ose-3.11-rpms
    baseurl=http://<repo-server>/repo/rhel-7-server-ose-3.11-rpms
    enabled=1
    gpgcheck=0
```

  * [ ] disable all repository

```
	subscription-manager repos --disable="*"
```
  * [ ] Cleanup all cache

```
	rm -rf /var/cache/yum
```
   * [ ] Testing Yum Repository Setup. The following command should list only 4 repositories.

```
	yum repolist
```
  * [ ] Copy /etc/yum.repos.d/ose.repo to all master and worker nodes to ensure they all can access the OpenShift RPM repository

```
    scp /etc/yum.repos.d/ose.repo  <host>:/etc/yum.repos.d/ose.repo
```

# 5. Setup local Docker registry server
This server will host the RedHat OpenShift images from registry.redhat.io for installation purpose in an airgaped environment. We will be using a docker container "regisry:2" for running the registry server. <i>This docker registry image need to be downloaded from a internet facing machine and transfer to Bastion Host</i>.
* [ ] Downloading docker registry image and test busybox image
This step need to be done on a internet facing system.

```        
    docker pull  docker.io/library/registry:2
    docker pull  docker.io/library/busybox:latest
    docker save -o registry.tar \
            docker.io/library/registry:2 \
            docker.io/library/busybox:latest
```
* [ ] Load Registry
```
	docker load -i registry2.tar
```

* [ ] Update /etc/containers/registry.conf, update registry.conf file to use your image registry. 
```
	[registries.insecure]
	registries = ["registry.ibmcloudpack.com:5000"]
```
* [ ] Start the local registry

```
	docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

* [ ] Testing the registry setup
```
    docker images
    docker tag docker.io/busybox  <<registry.ibmcloudpack.com>>:5000/busybox
    docker push <<registry.ibmcloudpack.com>>:5000/busybox
```

# 6. Load OpenShift 3.11 images into local docker registry
* [ ] Load openshift 3.11 images
```
    cd /ibm
    docker load -i ose3-images.tar
```

* [ ] Retag the images with your registry server
OpenShift images are by default prefixed with "registry.redhat.io". This need to be retaged with local docker registry server then pushed into local registry. Here is a simple script to automate the steps.
Create retag.sh. <b>Make sure, you update the script with your bastion host name</b>

```
    #!/bin/bash
    for i in `docker images | grep '^registry.redhat.io' | awk '{print $1":"$2}'`
    do
        b=`echo $i | sed 's/registry.redhat.io/<<bastion hostname>>:5000/g'`
        docker tag $i $b
        docker push $b
    done
```
Then Run the command:
```
	./ocp-docker-retag.sh	
```

* [ ] Updating the images with new tags, if needed example version v3.11.154
Create retag-v154.sh
``` 	 
     #!/bin/bash
	for i in `docker images | grep '^registry.redhat.io' | grep -i v3.11 | grep -v v3.11.146 | grep -v v3.11.135 | awk '{print $1":"$2}'`
	do
	    b=`echo $i | sed 's/registry.redhat.io/<bastion hostname>:5000/g'`.135
	    echo $b
	    docker tag $i $b
	    docker push $b
	done
```
and Run the retag-v154.sh

# 7. Setup NFS server for persistent storage for Cloud Pak for Data.
NFS server need to be setup in a machine with 1TB storage disk. In this example the work1 - Worker 3 are provisioned with additional 1TB disks. We are going to use <b>Work1</b> for running the local NFS server.

* [ ] Install nfs server
Log into worker1 and run the following command
```
    yum install -y nfs-utils
    systemctl enable rpcbind
    systemctl enable nfs-server
    systemctl start rpcbind
    systemctl start nfs-server
```

* [ ] Configure firewall ports

```
    firewall-cmd --zone=public --add-port=111/tcp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --add-port=111/udp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --add-port=2049/tcp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --add-port=2049/udp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --add-port=892/tcp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --add-port=662/udp --permanent
    firewall-cmd --reload
```

        
* [ ] Allow all nodes in the subnet to access it

```
    vi /etc/exports
```

Add the following line to allow all nodes in the cluster to access the NFS server for file sharing.

```    
    /data *(rw,sync,no_root_squash)
```

* [ ] Restart the nfs server

```    
    systemctl restart nfs-server
```

* [ ] Test the NFS mount on worker2
  Go to any of the worker nodes, example worker2

```
    mkdir /data
    mount -t nfs <worker1>:/data /data
    cd /data
```

Make sure this mount work correctly. After the test, umount the share
```
    umount /data
```

# 8. Preparing Cluster nodes
Below are the sequences of steps needed to prepare the worker, master and loadbalancer nodes before starting the openshift install. All these commands will be run from bastion Host.



* [ ] 1. Password less ssh from Bastion and all nodes
```
     ssh-keygen
     ssh-copy-id <all cluster nodes>
```

* [ ] 2. Install ansible on Bastion host
```
	yum install -y ansible
```
        ensure, it is installing the ansible version 2.6.19
```
        ansible --version
```
* [ ] 3. Install ansible-Openshift on Bastion Host only
```
	yum -y install openshift-ansible
```

* [ ] 4. Clocksync on all nodes
  - $ ansible-playbook -i ansible.os25 playbooks/clocksync.yml

* [ ] 3. Ipv4 forward
  - $ ansible-playbook -i ansible.os25 playbooks/ipforward.yml 

* [ ] 4. SELINUX enforced
  - $ ansible-playbook -i ansible.os25 playbooks/seenforce.yml 

* [ ] 5. Enable NetworkManager
  - $ ansible-playbook -i ansible.os25 playbooks/network-manager.yml 

* [ ] 6. Partition and Mount docker disk on /var/lib/docker
  - $ ansible-playbook -i ansible.os25 playbooks/docker-storage-mount.yml
  
  Here is a step to manually partition disk and mount to the file system.
  Create <b>partition_disk.sh</b>
  ```
      #!/bin/bash
	if [[ $# -ne 2 ]]; then
	    echo "Requires a disk name and mounted path name"
	    echo "$(basename $0) <disk> <path>"
	    exit 1
	fi
	set -e
	parted ${1} --script mklabel gpt
	parted ${1} --script mkpart primary '0%' '100%'
	mkfs.xfs -f -n ftype=1 ${1}1
	mkdir -p ${2}
	echo "${1}1       ${2}              xfs     defaults,noatime    1 2" >> /etc/fstab
	mount ${2}
	exit 0
   ```
   Then run the command for each partition on every nodes.
 
   ```
   sh partition_disk.sh /dev/<deviceName>	/<fileSystemName>

   example:
   sh partition_disk.sh /dev/sdb /var/lib/docker
   ```
  

* [ ] 7. Disable all RPM repos
  - $ ansible-playbook -i hosts playbooks/disable-redhat-repos.yml 


* [ ] 8. Copy ose.repo to all nodes
  - $ ansible-playbook -i hosts playbooks/yum-ose-repo.yml 

* [ ] 9. Install preinstall packages
  - $ ansible-playbook -i hosts playbooks/preinstallpackages.yml

* [ ] 10. Reboot all cluster nodes
These nodes need to be restarted to make SELINUX setting to be enfective, 
  - $ ansible-playbook -i hosts playbooks/reboot-cluster.yml   

# 9. Load Balancer setup
External access to OpenShift cluster are achieved using a load balancer. In high availability scenario, external load balancer server as single entry point to OpenShift cluster. Two different load balancer are used, one for control plane , separate one for application workloads running on OpenShift. In POC like, environment users usually setup a small Virtual machine to act as a loadbalancer.

Please refer RedHat documentation for more details https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html/configuring_clusters/install-config-routing-from-edge-lb


![alt text][logo]

[logo]: https://github.com/ekambaraml/openshift311-airgap/blob/master/OpenShift-withLB.png "How Load Balancer is used"

# 10. Create OpenShift Inventory file
Inventory file is created based your cluster configuration. Sample inventory files are attached in this repo. for reference.

NFS as a Storage: https://github.com/ekambaraml/openshift311-airgap/blob/master/inventory.nfs
GlusterFS as a storage: https://github.com/ekambaraml/openshift311-airgap/blob/master/inventory.glusterfs

# 11. Deploy OpenShift 3.11
On Bastion host, run the following commands to deploy openshift 3.11 
```
	$ ansible-playbook -i  inventory /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
	$ ansible-playbook -i  inventory /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

# 12. Validating successful OpenShift install

* [ ] 1. Test the cluster and pods are up and Running
```
    oc get nodes
    oc get pods –all-namespaces

```
* [ ] Find URL of the openshift admin console
```
	oc get routes -n openshift-console | grep console
	console   console.apps.examples.com             console    https     reencrypt/Redirect   None
```
* [ ] Open the url https://console.apps.examples.com in Firefox or chrome browser
Default admin user/password is setup to "ocadmin" and "ocadmin".   Change the password after you validate the cluster is fully ready.



