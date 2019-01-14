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

	$ kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=ocid1.user.oc1..aaaaaaaa724gophmrcxxrzg3utunh3rg5ieeyyrwuqfetixdb3mhzesxmdbq
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

#### Setup worker node ####

Before the deployment of WebLogic Operator and Domain you need to setup the worker node. Operator v1.1 requires persistent volume configured. In this case this means a folder creation in advance on the node.

Also to make the deployment smoother you need to pull the official Oracle WebLogic image from Docker Store before the deployment.

Log in using `ssh` and the public IP address (!Don't forget to modify the IP address to your node's address) to the worker node:

	$ ssh -i ~/.ssh/oow.id_rsa opc@130.61.58.999
	Oracle Linux Server 7.5
	Last login: Sun Jan  6 23:27:24 2019 from 46.139.97.155
	[opc@oke-c2tcyzzmrqw-n2temrqgyyt-s54zyw2hoiq-0 ~]$

Change to *root* user.

	[opc]$ sudo su -

Create `/scratch/external-domain-home/pv001` shared directory for domain binaries.

	[root]$ mkdir -m 777 -p /scratch/external-domain-home/pv001
	[root]$ chown -R opc:opc /scratch

Type **exit** to end *root* session.

	[root]$ exit

Last step to prepare this node is to pull the `store/oracle/weblogic:12.2.1.3` image from Docker store. Before issue pull command you have to login to Docker.

	$ docker login
	Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
	Username: <YOUR_DOCKER_USERNAME>
	Password: <YOUR_DOCKER_PASSWORD>
	Login Succeeded

After the successful login pull the official WebLogic Docker image:

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

The worker node preparation is done type **exit** again to end the *ssh* session.

	[opc]$ exit

#### Prepare the operator and domain configuration YAML files ####

Customize the parameters in the input YAML files for the WebLogic cluster and WebLogic Operator. The input YAML files and scripts provided in the [WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator) project.

The WebLogic Kubernetes Operator source repository can be found in the `/u01/content/weblogic-kubernetes-operator`. Open for edit the `create-weblogic-operator-inputs.yaml` parameter file using File Browser or using the following command:

	gedit /u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-operator-inputs.yaml &

Modify the following parameters in the *create-weblogic-operator-inputs.yaml* input file:

| Parameter name                        | Value                                                                  | Note                                                                       |
|---------------------------------------|------------------------------------------------------------------------|----------------------------------------------------------------------------|
| targetNamespaces                      | domain1                                                                |                                                                            |
| weblogicOperatorImage                 | oracle/weblogic-kubernetes-operator:1.0 |  | |


Save the changes. Open for edit the `create-weblogic-domain-inputs.yaml` parameter file using File Browser or using the following command:

	gedit /u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-domain-inputs.yaml &

and modify the following parameters in the *create-weblogic-domain-inputs.yaml* input file:

| Parameter name            | Value                               | Note |
|---------------------------|-------------------------------------|------|
| domainUID                 | domain1                             | uncomment if neccessary |
| weblogicDomainStoragePath | /scratch/external-domain-home/pv001 | uncomment if neccessary |
| exposeAdminT3Channel      | true                                |      |
| exposeAdminNodePort       | true                                |      |
| namespace                 | domain1                             |      |
|loadBalancer               | TRAEFIK                             | it should be the default |

Save the changes.

#### Deploy WebLogic Kubernetes Operator and WebLogic Domain ####

Open a terminal and create output directory for the operator and domain scripts.

	mkdir -p /u01/weblogic-output-directory

First run the create operator script, pointing it at your inputs file and the output directory.

	/u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-operator.sh -i /u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-operator-inputs.yaml -o /u01/weblogic-output-directory
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

	/u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-domain.sh -i /u01/content/weblogic-kubernetes-operator/kubernetes/create-weblogic-domain-inputs.yaml -o /u01/weblogic-output-directory
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

You have to see four running pods similar to the result above. If you don't see all the running pods please wait and check periodically. The whole domain deployment may take up to 3-4 minutes depending on the compute shape.

The WebLogic Administration server console is exposed to the external world using NodePort. To get the external IP address of the node where the admin server pod is deployed execute the following command which returns external (public) IP addresses belong to pods:

	kubectl get pod -n domain1 -o wide
	NAME                                         READY     STATUS    RESTARTS   AGE       IP           NODE
	domain1-admin-server                         1/1       Running   0          32m       10.244.2.5   130.61.58.206
	domain1-cluster-1-traefik-55cbccb8dd-srpcb   1/1       Running   0          32m       10.244.2.4   130.61.58.206
	domain1-managed-server1                      0/1       Running   4          29m       10.244.0.6   130.61.58.206
	domain1-managed-server2                      0/1       Running   0          29m       10.244.1.4   130.61.58.206

Find the Node IP address belongs to the *domain1-admin-server* pod.

To get the port assignment retrieve the admin server service information. Execute the following:

	kubectl get services -n domain1
	NAME                                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
	domain1-admin-server                        NodePort    10.96.13.10    <none>        7001:30701/TCP    8m
	domain1-admin-server-extchannel-t3channel   NodePort    10.96.69.58    <none>        30012:30012/TCP   6m
	domain1-cluster-1-traefik                   NodePort    10.96.97.188   <none>        80:30305/TCP      8m
	domain1-cluster-1-traefik-dashboard         NodePort    10.96.176.74   <none>        8080:30315/TCP    8m
	domain1-cluster-cluster-1                   ClusterIP   10.96.36.241   <none>        8001/TCP          6m
	domain1-managed-server1                     ClusterIP   10.96.95.211   <none>        8001/TCP          6m
	domain1-managed-server2                     ClusterIP   10.96.130.75   <none>        8001/TCP          6m

Find the port mapping belongs to the *domain1-admin-server* service. Note the second part of the mapping. It should be 30701 if you haven't changed the configuration. Construct the Administration Console's url and open in a browser:

**http://<LOADBALANCER\_POD\_NODE\_PUBLIC\_IP\_ADDRESS>:30701/console**

![alt text](images/deploy.weblogic/11.weblogic.console.url.png)

Enter the credentials defined in the secret *domain1-weblogic-credentials* previously (username:**weblogic**, password:**welcome1**). Click login.

![alt text](images/deploy.weblogic/12.weblogic.admin.console.png)

Congratulations! You have successfully deployed WebLogic domain on (OKE) Kubernetes cluster.
