# Setup WebLogic Domain on Kubernetes #

Oracle provides WebLogic Kubernetes Operator to run WebLogic clusters in production or development mode and on Kubernetes clusters on-premises or in the cloud. The scope of this tutorial to show the steps to run a WebLogic cluster using the Oracle Cloud Infrastructure (OCI) Container Engine for Kubernetes (OKE).

This tutorial is based on the official [Oracle WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator/blob/master/site/creating-domain.md) domain creation guide.

### Prerequisites ###

- [Oracle Cloud Infrastructure](https://cloud.oracle.com/en_US/cloud-infrastructure) enabled account.
- Running [Container Engine for Kubernetes (OKE)](setup.oke.md) cluster.
- [Oracle WebLogic Kubernetes Operator](create.weblogic.operator.md) image uploaded to Container Registry (OCIR).
- Desktop with `kubectl` installed. `kubectl` has to be configured to access to the Kubernetes Cluster.

## Prepare the WebLogic Kubernetes Operator Environment

#### Set up the RBAC policy for the OKE cluster ####

In order to have permission to access the Kubernetes cluster, you need to authorize your OCI account as a *cluster-admin* on the OCI Container Engine for Kubernetes cluster. This will require your OCID, which is available on the OCI console page, under your user settings. Click **Show** next to the OCID label and part of the value.

![alt text](images/deploy.weblogic/01.user.ocid.information.png)

Then execute the role binding command using your OCID:

	kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=<YOUR_USER_OCID>

For example:

	$ kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=ocid1.user.oc1..aaaaaaaap6s7dzkleipjc7cvidmn64sk5o6c33dbfrcq2sbj7vzi2f3ucola
	clusterrolebinding "my-cluster-admin-binding" created

#### Accept Licence Agreement to use `store/oracle/weblogic:12.2.1.3` image from Docker Store ####

If you have not used the base image [`store/oracle/weblogic:12.2.1.3`](https://store.docker.com/images/oracle-weblogic-server-12c) before, you will need to visit the [Docker Store web interface](https://store.docker.com/images/oracle-weblogic-server-12c) and accept the license agreement before the Docker Store will give you permission to pull that image.

Open [https://store.docker.com/images/oracle-weblogic-server-12c](https://store.docker.com/images/oracle-weblogic-server-12c) in a new browser and click **Log In**.

![alt text](images/deploy.weblogic/05.docker.store.weblogic.png)

Enter your account details and click **Login**.

![](images/build.operator/02.docker.store.login.png)

Click **Proceed to Checkout**.

![alt text](images/deploy.weblogic/07.docker.store.weblogic.checkout.png)

Complete your contact information and accept agreements. Click **Get Content**.

![alt text](images/deploy.weblogic/08.docker.store.weblogic.get.content.png)

Now you are ready to pull the  image on docker enabled host after authenticating yourself in Docker Hub using your Docker Hub credentials.

![alt text](images/deploy.weblogic/09.docker.store.weblogic.png)

#### Set up the NFS server ####

The WebLogic domain must be installed into the folder that will be mounted as /shared/domain

At this time, the WebLogic on Kubernetes domain created by the WebLogic Server Kubernetes Operator, requires a shared file system to store the WebLogic domain configuration, which MUST be accessible from all the pods across the nodes. As a workaround, you need to configure an NFS server on one node and share the file system across all the nodes.

**Note**: Currently, we recommend that you use NFS version 3.0 for running WebLogic Server on OCI Container Engine for Kubernetes.

##### Retrieve node's public and private IP addresses #####

Enter the following `kubectl` command to get the public IP addresses of the nodes:

	$ kubectl get node

The output of the command will display the nodes, similar to the following:

	NAME              STATUS    ROLES     AGE       VERSION
	129.146.103.156   Ready     node      9d        v1.9.7
	129.146.133.192   Ready     node      9d        v1.9.7
	129.146.166.187   Ready     node      9d        v1.9.7

To get private IP addresses you need to `ssh` into every node. This requires your private (RSA) key pair of the public key what you defined during OKE creation. Execute the following command for every node:

**ssh -i <YOUR\_RSA\_PRIVATE\_KEY\_LOCATION> opc@<PUBLIC\_IP\_OF\_NODE>**

For example:

	$ ssh -i ~/.ssh/id_rsa opc@129.146.103.156
	The authenticity of host '129.146.103.156 (129.146.103.156)' can't be established.
	ECDSA key fingerprint is SHA256:DjSi+nQJRgnuapgQp/zA9HisHzHjbev+jnjrBluAboo.
	ECDSA key fingerprint is MD5:8f:80:de:8b:0c:6e:29:c3:9c:bf:ee:6a:79:63:38:39.
	Are you sure you want to continue connecting (yes/no)? yes
	Warning: Permanently added '129.146.103.156' (ECDSA) to the list of known hosts.
	Oracle Linux Server 7.4
	Last login: Sat Jun 16 10:51:32 2018 from 86.59.134.172
	$

Retrieve private IP address executing this command:

	$ ip addr | grep ens3
	2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP qlen 1000
    	inet 10.0.12.4/24 brd 10.0.12.255 scope global dynamic ens3

Find the inet value. For example the private address in the example above: 10.0.12.4

Repeat the query for each node. Note the IP addresses for easier usage:

| Nodes:             | Public IP       | Private IP |
|--------------------|-----------------|------------|
| Node1 - NFS Server | 129.146.103.156 | 10.0.12.4  |
| Node2              | 129.146.133.192 | 10.0.11.3  |
| Node3              | 129.146.166.187 | 10.0.10.3  |


###### Node1 - NFS server ######

Log in using `ssh` to **Node1**, and set up NFS Server:

	$ ssh -i ~/.ssh/id_rsa opc@129.146.103.156
	Oracle Linux Server 7.4
	Last login: Sat Jun 16 10:51:32 2018 from 86.59.134.172

Change to *root* user.

	[opc]$ sudo su -

Create `/scratch/external-domain-home/pv001` shared directory for domain binaries.

	[root]$ mkdir -m 777 -p /scratch/external-domain-home/pv001
	[root]$ chown -R opc:opc /scratch

Modify `/etc/exports` to enable Node2, Node3 access to Node1 NFS server.

	[root]$ vi /etc/exports

---

**NOTE!** By the default the NFS server has to be installed but stopped on OKE node. If the NFS is not installed on node then run `yum install -y nfs-utils` first as super user.

---

Add private IP addresses of Node2 and Node3.

	/scratch 10.0.11.3(rw)
	/scratch 10.0.10.3(rw)
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	"/etc/exports" 2L, 46C

Save the changes and restart NFS service.

	[root]$ service nfs-server start

Type **exit** to end *root* session.

	[root]$ exit

Last step to prepare this node is to pull the `store/oracle/weblogic:12.2.1.3` image from Docker store. Before issue pull command you have to login to Docker.

	$ docker login
	Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
	Username: <YOUR_DOCKER_USERNAME>
	Password: <YOUR_DOCKER_PASSWORD>
	Login Succeeded
	[opc]$ docker pull store/oracle/weblogic:12.2.1.3
	Trying to pull repository docker.io/store/oracle/weblogic ...
	12.2.1.3: Pulling from docker.io/store/oracle/weblogic
	9fd8609e6e4d: Pull complete
	eac7b4a33e34: Pull complete
	b6f7d13c859b: Pull complete
	e0ca246b2272: Pull complete
	7ba4d6bfba43: Pull complete
	5e3b8c4731f0: Pull complete
	97623ceb6339: Pull complete
	Digest: sha256:4c7ce451c093329784a2808a55cd4fc4f1e93d8444b8492a24148d283936add9
	Status: Downloaded newer image for docker.io/store/oracle/weblogic:12.2.1.3

Node1 preparation is done type **exit** again to end the *ssh* session.

	[opc]$ exit

###### Node2 - NFS client ######

Log in using `ssh` to **Node2** and configure NFS client:

	$ ssh -i ~/.ssh/id_rsa opc@129.146.133.192
	Oracle Linux Server 7.4
	Last login: Sat Jun 16 10:51:32 2018 from 86.59.134.172

Change to *root* user.

	[opc]$ sudo su -

Create `/scratch` shared directory.

	[root]$ mkdir /scratch

Edit the `/etc/fstab` file.

	[root]$ vi /etc/fstab

Add the internal IP address and parameters of **Node1 - NFS server**. Append as last row.

	#
	# /etc/fstab
	# Created by anaconda on Fri Feb  9 01:25:44 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	UUID=7247af6c-4b59-4934-a6be-a7929d296d83 /                       xfs     defaults,_netdev,_netdev 0 0
	UUID=897D-798C          /boot/efi               vfat    defaults,uid=0,gid=0,umask=0077,shortname=winnt,_netdev,_netdev,x-initrd.mount 0 0
	######################################
	## ORACLE BARE METAL CLOUD CUSTOMERS
	##
	## If you are adding an iSCSI remote block volume to this file you MUST
	## include the '_netdev' mount option or your instance will become
	## unavailable after the next reboot.
	##
	## Example:
	## /dev/sdb /data1  ext4    defaults,noatime,_netdev    0   2
	##
	## More information:
	## https://docs.us-phoenix-1.oraclecloud.com/Content/Block/Tasks/connectingtoavolume.htm
	##
	10.0.12.4:/scratch /scratch  nfs nfsvers=3 0 0
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	"/etc/fstab" 24L, 957C

Save changes. Mount the shared `/scratch` directory.

	[root]$ mount /scratch

Restart the NFS service.

	[root]$ service nfs-server restart

Type **exit** to end *root* session.

		[root]$ exit

Last step to prepare this node is to pull the `store/oracle/weblogic:12.2.1.3` image from Docker store. Before issue pull command you have to login to Docker.

	$ docker login
	Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
	Username: <YOUR_DOCKER_USERNAME>
	Password: <YOUR_DOCKER_PASSWORD>
	Login Succeeded
	[opc]$ docker pull store/oracle/weblogic:12.2.1.3
	Trying to pull repository docker.io/store/oracle/weblogic ...
	12.2.1.3: Pulling from docker.io/store/oracle/weblogic
	9fd8609e6e4d: Pull complete
	eac7b4a33e34: Pull complete
	b6f7d13c859b: Pull complete
	e0ca246b2272: Pull complete
	7ba4d6bfba43: Pull complete
	5e3b8c4731f0: Pull complete
	97623ceb6339: Pull complete
	Digest: sha256:4c7ce451c093329784a2808a55cd4fc4f1e93d8444b8492a24148d283936add9
	Status: Downloaded newer image for docker.io/store/oracle/weblogic:12.2.1.3

Node2 preparation is done type **exit** again to end the *ssh* session.

	[opc]$ exit

###### Node3 - NFS client ######

Log in using `ssh` to **Node3** and configure NFS client:

	$ ssh -i ~/.ssh/id_rsa opc@129.146.166.187
	Oracle Linux Server 7.4
	Last login: Sat Jun 16 10:51:32 2018 from 86.59.134.172

Change to *root* user.

	[opc]$ sudo su -

Create `/scratch` shared directory.

	[root]$ mkdir /scratch

Edit the `/etc/fstab` file.

	[root]$ vi /etc/fstab

Add the internal IP address and parameters of **Node1 - NFS server**. Append as last row.

	#
	# /etc/fstab
	# Created by anaconda on Fri Feb  9 01:25:44 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	UUID=7247af6c-4b59-4934-a6be-a7929d296d83 /                       xfs     defaults,_netdev,_netdev 0 0
	UUID=897D-798C          /boot/efi               vfat    defaults,uid=0,gid=0,umask=0077,shortname=winnt,_netdev,_netdev,x-initrd.mount 0 0
	######################################
	## ORACLE BARE METAL CLOUD CUSTOMERS
	##
	## If you are adding an iSCSI remote block volume to this file you MUST
	## include the '_netdev' mount option or your instance will become
	## unavailable after the next reboot.
	##
	## Example:
	## /dev/sdb /data1  ext4    defaults,noatime,_netdev    0   2
	##
	## More information:
	## https://docs.us-phoenix-1.oraclecloud.com/Content/Block/Tasks/connectingtoavolume.htm
	##
	10.0.12.4:/scratch /scratch  nfs nfsvers=3 0 0
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	~                                                                                                                                                                               
	"/etc/fstab" 24L, 957C

Save changes. Mount the shared `/scratch` directory.

	[root]$ mount /scratch

Restart the NFS service.

	[root]$ service nfs-server restart

Type **exit** to end *root* session.

	[root]$ exit

Last step to prepare this node is to pull the `store/oracle/weblogic:12.2.1.3` image from Docker store. Before issue pull command you have to login to Docker.

	$ docker login
	Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
	Username: <YOUR_DOCKER_USERNAME>
	Password: <YOUR_DOCKER_PASSWORD>
	Login Succeeded
	[opc]$ docker pull store/oracle/weblogic:12.2.1.3
	Trying to pull repository docker.io/store/oracle/weblogic ...
	12.2.1.3: Pulling from docker.io/store/oracle/weblogic
	9fd8609e6e4d: Pull complete
	eac7b4a33e34: Pull complete
	b6f7d13c859b: Pull complete
	e0ca246b2272: Pull complete
	7ba4d6bfba43: Pull complete
	5e3b8c4731f0: Pull complete
	97623ceb6339: Pull complete
	Digest: sha256:4c7ce451c093329784a2808a55cd4fc4f1e93d8444b8492a24148d283936add9
	Status: Downloaded newer image for docker.io/store/oracle/weblogic:12.2.1.3

Node3 preparation is done type **exit** again to end the *ssh* session.

	[opc]$ exit

#### Modify the operator and domain configuration YAML files ####

Final steps are to customize the parameters in the input YAML files for the WebLogic cluster and WebLogic Operator. The input YAML files and scripts provided in the [WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator) project.

Most likely you already cloned locally the WebLogic Kubernetes Operator source repository to build the operator image. If not then first clone locally. Using the following command the sources will be cloned into `weblogic-kubernetes-operator` folder in the current directory.

	git clone https://github.com/oracle/weblogic-kubernetes-operator.git
	Cloning into 'weblogic-kubernetes-operator'...
	remote: Counting objects: 17144, done.
	remote: Compressing objects: 100% (250/250), done.
	remote: Total 17144 (delta 149), reused 288 (delta 92), pack-reused 16757
	Receiving objects: 100% (17144/17144), 10.90 MiB | 1.99 MiB/s, done.
	Resolving deltas: 100% (10803/10803), done.

Change directory to `weblogic-kubernetes-operator/kubernetes` where the parameters input files located.

	cd weblogic-kubernetes-operator/kubernetes

Open and modify the following parameters in the *create-weblogic-operator-inputs.yaml* input file:

| Parameter name                        | Value                                                                  | Note                                                                       |
|---------------------------------------|------------------------------------------------------------------------|----------------------------------------------------------------------------|
| targetNamespaces                      | domain1                                                                |                                                                            |
| weblogicOperatorImage                 | oracle/weblogic-kubernetes-operator:1.0 |  | |


Save the changes. Open and modify the following parameters in the *create-weblogic-domain-inputs.yaml* input file:

| Parameter name            | Value                               | Note |
|---------------------------|-------------------------------------|------|
| domainUID                 | domain1                             | uncomment if neccessary |
| weblogicDomainStoragePath | /scratch/external-domain-home/pv001 | uncomment if neccessary |
| t3PublicAddress           | 0.0.0.0                             |      |
| exposeAdminT3Channel      | true                                |      |
| exposeAdminNodePort       | true                                |      |
| namespace                 | domain1                             |      |
|loadBalancer               | TRAEFIK                             | it should be the default |

Save the changes.

#### Deploy WebLogic Kubernetes Operator and WebLogic Domain ####

Create output directory for the operator and domain scripts.

	mkdir -p /PATH_TO/weblogic-output-directory

First run the create operator script, pointing it at your inputs file and the output directory. The best to execute in the locally cloned `weblogic-kubernetes-operator/kubernetes` folder:

	./create-weblogic-operator.sh -i create-weblogic-operator-inputs.yaml -o /PATH_TO/weblogic-output-directory
	Input parameters being used
	export version="create-weblogic-operator-inputs-v1"
	export serviceAccount="weblogic-operator"
	export namespace="weblogic-operator"
	export targetNamespaces="domain1"
	export weblogicOperatorImage="peternagy/weblogic-kubernetes-operator:e77427ed82d2d3a1cb5a21e0e56720e2d24076c3"
	export weblogicOperatorImagePullPolicy="IfNotPresent"
	export weblogicOperatorImagePullSecretName="dockersecret"
	export externalRestOption="SELF_SIGNED_CERT"
	export externalRestHttpsPort="31001"
	export externalSans="IP:129.146.120.224"
	export remoteDebugNodePortEnabled="false"
	export internalDebugHttpPort="30999"
	export externalDebugHttpPort="30999"
	export javaLoggingLevel="FINER"
	export elkIntegrationEnabled="false"

	The WebLogic operator REST interface is externally exposed using a generated self-signed certificate that contains the customer-provided list of subject alternative names.
	Checking to see if the secret dockersecret exists in namespace weblogic-operator
	/u01/content/weblogic-kubernetes-operator/kubernetes/internal
	Generating a self-signed certificate for the operator's internal https port with the subject alternative names DNS:internal-weblogic-operator-svc,DNS:internal-weblogic-operator-svc.weblogic-operator,DNS:internal-weblogic-operator-svc.weblogic-operator.svc,DNS:internal-weblogic-operator-svc.weblogic-operator.svc.cluster.local
	Generating a self-signed certificate for the operator's external ssl port with the subject alternative names IP:129.146.120.224
	Generating /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator.yaml
	Running the weblogic operator security customization script
	...
	Generating YAML script /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator-security.yaml to create WebLogic Operator security configuration...
	Create the WebLogic Operator Security configuration using kubectl as follows: kubectl create -f /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator-security.yaml
	Ensure you start the API server with the --authorization-mode=RBAC option.
	Checking to see if the namespace weblogic-operator already exists
	The namespace weblogic-operator already exists
	Checking the target namespace domain1
	Checking to see if the namespace domain1 already exists
	The namespace domain1 already exists
	Checking to see if the service account weblogic-operator already exists
	The service account weblogic-operator already exists
	Applying the generated file /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator-security.yaml
	namespace "weblogic-operator" configured
	serviceaccount "weblogic-operator" unchanged
	clusterrole "weblogic-operator-cluster-role" configured
	clusterrole "weblogic-operator-cluster-role-nonresource" configured
	clusterrolebinding "weblogic-operator-operator-rolebinding" configured
	clusterrolebinding "weblogic-operator-operator-rolebinding-nonresource" configured
	clusterrolebinding "weblogic-operator-operator-rolebinding-discovery" configured
	clusterrolebinding "weblogic-operator-operator-rolebinding-auth-delegator" configured
	clusterrole "weblogic-operator-namespace-role" configured
	rolebinding "weblogic-operator-rolebinding" configured
	Checking the cluster role weblogic-operator-namespace-role was created
	Checking role binding weblogic-operator-rolebinding was created for each target namespace
	Checking role binding weblogic-operator-rolebinding for namespace domain1
	Checking the cluster role weblogic-operator-cluster-role was created
	Checking the cluster role bindings weblogic-operator-operator-rolebinding were created
	Applying the file /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator.yaml
	configmap "weblogic-operator-cm" configured
	secret "weblogic-operator-secrets" configured
	deployment "weblogic-operator" configured
	service "external-weblogic-operator-svc" unchanged
	service "internal-weblogic-operator-svc" unchanged
	Waiting for operator deployment to be ready...
	status is 1, iteration 1 of 10
	Checking the operator labels
	Checking the operator pods
	Checking the operator Pod status

	The Oracle WebLogic Server Kubernetes Operator is deployed, the following namespaces are being managed: domain1

	The following files were generated:
	  /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/create-weblogic-operator-inputs.yaml
	  /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator.yaml
	  /u01/content/weblogic-output-directory/weblogic-operators/weblogic-operator/weblogic-operator-security.yaml

	Completed

Check the pod status in *weblogic-operator* namespace:

	kubectl get pod -n weblogic-operator
	NAME                                 READY     STATUS    RESTARTS   AGE
	weblogic-operator-58d944d448-jxbph   1/1       Running   0          1m

Before execute the last domain creation script you have to create the WebLogic admin credentials. The username and password credentials for access to the Administration Server must be stored in a Kubernetes secret in the same namespace that the domain will run in. The script does not create the secret in order to avoid storing the credentials in a file. To create the secret for this demo, issue the following command (you can change the password below, but don't forget later):

	kubectl -n domain1 create secret generic domain1-weblogic-credentials --from-literal=username=weblogic --from-literal=password=welcome1

Now execute a similar command for the WebLogic domain creation:

	./create-weblogic-domain.sh -i create-weblogic-domain-inputs.yaml -o /PATH_TO/weblogic-output-directory
	Input parameters being used
	export version="create-weblogic-domain-inputs-v1"
	export adminPort="7001"
	export adminServerName="admin-server"
	export domainName="base_domain"
	export domainUID="domain1"
	export clusterType="DYNAMIC"
	export startupControl="AUTO"
	export clusterName="cluster-1"
	export configuredManagedServerCount="2"
	export initialManagedServerReplicas="2"
	export managedServerNameBase="managed-server"
	export managedServerPort="8001"
	export weblogicDomainStorageType="HOST_PATH"
	export weblogicDomainStoragePath="/scratch/external-domain-home/pv001"
	export weblogicDomainStorageReclaimPolicy="Retain"
	export weblogicDomainStorageSize="10Gi"
	export productionModeEnabled="true"
	export weblogicCredentialsSecretName="domain1-weblogic-credentials"
	export t3ChannelPort="30012"
	export t3PublicAddress="kubernetes"
	export exposeAdminT3Channel="false"
	export adminNodePort="30701"
	export exposeAdminNodePort="false"
	export namespace="domain1"
	export loadBalancer="TRAEFIK"
	export loadBalancerAppPrepath="/"
	export loadBalancerWebPort="30305"
	export loadBalancerDashboardPort="30315"
	export javaOptions="-Dweblogic.StdoutDebugEnabled=false"

	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-pv.yaml
	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-pvc.yaml
	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/create-weblogic-domain-job.yaml
	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/domain-custom-resource.yaml
	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-traefik-cluster-1.yaml
	Generating /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-traefik-security-cluster-1.yaml
	Checking to see if the secret domain1-weblogic-credentials exists in namespace domain1
	Checking if the persistent volume domain1-weblogic-domain-pv exists
	No resources found.
	The persistent volume domain1-weblogic-domain-pv does not exist
	Creating the persistent volume domain1-weblogic-domain-pv
	persistentvolume "domain1-weblogic-domain-pv" created
	Checking if the persistent volume domain1-weblogic-domain-pv is Available
	Checking if the persistent volume claim domain1-weblogic-domain-pvc in namespace domain1 exists
	No resources found.
	The persistent volume claim domain1-weblogic-domain-pvc does not exist in namespace domain1
	Creating the persistent volume claim domain1-weblogic-domain-pvc
	persistentvolumeclaim "domain1-weblogic-domain-pvc" created
	Checking if the persistent volume domain1-weblogic-domain-pv is Bound
	Checking if object type job with name domain1-create-weblogic-domain-job exists
	No resources found.
	Creating the domain by creating the job /u01/content/weblogic-output-directory/weblogic-domains/domain1/create-weblogic-domain-job.yaml
	configmap "domain1-create-weblogic-domain-job-cm" created
	job "domain1-create-weblogic-domain-job" created
	Waiting for the job to complete...
	status on iteration 1 of 20
	pod domain1-create-weblogic-domain-job-jjssn status is ContainerCreating
	Error from server (BadRequest): container "create-weblogic-domain-job" in pod "domain1-create-weblogic-domain-job-jjssn" is waiting to start: ContainerCreating
	status on iteration 2 of 20
	pod domain1-create-weblogic-domain-job-jjssn status is Running
	status on iteration 3 of 20
	pod domain1-create-weblogic-domain-job-jjssn status is Completed
	Setting up traefik security
	clusterrole "domain1-cluster-1-traefik" created
	clusterrolebinding "domain1-cluster-1-traefik" created
	Checking the cluster role domain1-cluster-1-traefik was created
	Checking the cluster role binding domain1-cluster-1-traefik was created
	Deploying traefik
	serviceaccount "domain1-cluster-1-traefik" created
	deployment "domain1-cluster-1-traefik" created
	configmap "domain1-cluster-1-traefik-cm" created
	service "domain1-cluster-1-traefik" created
	service "domain1-cluster-1-traefik-dashboard" created
	Checking traefik deployment
	Checking the traefik service account
	Checking traefik service
	Creating the domain custom resource using /u01/content/weblogic-output-directory/weblogic-domains/domain1/domain-custom-resource.yaml
	domain "domain1" created
	Checking the domain custom resource was created

	Domain base_domain was created and will be started by the WebLogic Kubernetes Operator

	The load balancer for cluster 'cluster-1' is available at http://c2tqzjtmmyd.us-phoenix-1.clusters.oci.oraclecloud.com:30305/ (add the application path to the URL)
	The load balancer dashboard for cluster 'cluster-1' is available at http://c2tqzjtmmyd.us-phoenix-1.clusters.oci.oraclecloud.com:30315

	The following files were generated:
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/create-weblogic-domain-inputs.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-pv.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-pvc.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/create-weblogic-domain-job.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/domain-custom-resource.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-traefik-security-cluster-1.yaml
	  /u01/content/weblogic-output-directory/weblogic-domains/domain1/weblogic-domain-traefik-cluster-1.yaml

	Completed

To check the status of the WebLogic cluster, run this command:

	kubectl get pod -n domain1
	NAME                                         READY     STATUS    RESTARTS   AGE
	domain1-admin-server                         1/1       Running   0          9m
	domain1-cluster-1-traefik-778bc994f7-7mjw6   1/1       Running   0          9m
	domain1-managed-server1                      1/1       Running   0          7m
	domain1-managed-server2                      1/1       Running   0          7m

You have to see four running pods similar to the result above.

The WebLogic Administration server console is exposed to the external world using NodePort. To get the external IP address of the node where the admin server pod is deployed execute the following command which returns external (public) IP addresses belong to pods:

	$ kubectl get pod -n domain1 -o wide
	NAME                                         READY     STATUS    RESTARTS   AGE       IP           NODE
	domain1-admin-server                         1/1       Running   0          32m       10.244.2.5   129.146.133.192
	domain1-cluster-1-traefik-55cbccb8dd-srpcb   1/1       Running   0          32m       10.244.2.4   129.146.133.192
	domain1-managed-server1                      0/1       Running   4          29m       10.244.0.6   129.146.88.147
	domain1-managed-server2                      0/1       Running   0          29m       10.244.1.4   129.146.40.233

Find the Node IP address belongs to the loadbalancer (*domain1-cluster-1-traefik-xxxxxxx*) pod. If you haven't changed the domain configuration then the console port is 30701. Construct the Administration Console's url and open in a browser:

**http://<LOADBALANCER\_POD\_NODE\_PUBLIC\_IP\_ADDRESS>:30701/console**

![alt text](images/deploy.weblogic/11.weblogic.console.url.png)

Enter the credentials defined in the secret *domain1-weblogic-credentials* previously. Click login.

![alt text](images/deploy.weblogic/12.weblogic.admin.console.png)

Congratulations! You have successfully deployed WebLogic domain on (OKE) Kubernetes cluster.
