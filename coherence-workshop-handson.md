# Coherence on OKE Hands On Lab

## Objective

This lab will demonstrate the steps required to get Oracle Coherence up and running on Oracle Container Engine (OKE) using the Coherence Operator. It will then explore some of the advantages of running Coherence on Kubernetes on OCI. 

## Requirements

A username and password for the OCI tenancy used for the lab.

Details of the two Oracle Container Engine (OKE) clusters that will be used for the lab. 

## Establish access to London Cluster

This lab will use two OKE clusters, the first on the London region and the second in the Frankfurt region. We will interact with the OKE clusters via standard Kubernetes tools like kubectl and helm. Fortunately OCI provides a [Cloud Shell](https://docs.cloud.oracle.com/en-us/iaas/Content/API/Concepts/cloudshellintro.htm) environment with these tools already installed that we can use as a virtual bastion host. 

To get started locate the OCI username and password assigned to you and open the OCI Console at https://console.uk-london-1.oraclecloud.com/

//TODO - Screenshots of login process

We will now configure kubectl in cloud shell to work with the London OKE cluster. Click on the "hamburger" icon in the top left to open the services menu and locate "Developer Services" and then the "Kubernetes Clusters".

![image-20200929084456958](image-20200929084456958.png)

On the OKE clusters homepage locate the cluster assigned to you. Ensure you are in the correct compartment as specified in the student details by checking the compartment drop down in the left of the screen. Click your assigned OKE cluster to view it's details, then select the blue "Access Cluster" button. 

![image-20200929090516869](image-20200929090516869.png)

Configuring kubectl is a case of just following the instructions on the screen. First press the "Launch Cloud Shell" button, after a few moments a terminal will launch at the bottom of your browser window. Then copy the oci cli command and paste it into the cloud shell terminal. Your cloud shell is pre-authenticated against your OCI account so no credentials are needed. The command will copy the kube config file to the standard location at ~/.kube/config. You should now be able to query your OKE cluster by running using the pre-installed kubectl command:

```bash
kubectl get nodes -o wide
```

in the cloud shell window.


![image-20200929091157353](image-20200929091157353.png)

You can now close the "Access Your Cluster" window.

We will now rename the context created to allow us to switch between the London and Frankfurt clusters later in the lab. Query the current context with the command:

```
$ kubectl config get-contexts
CURRENT   NAME                  CLUSTER               AUTHINFO           NAMESPACE
*         context-cqtsyzvge2w   cluster-cqtsyzvge2w   user-cqtsyzvge2w  
```

Rename the context to lhr which should prove to be a little more memorable!
The following command will need to reflect the context name returned from the command above.

```
$ kubectl config rename-context <CONTEXT_NAME> lhr
Context "context-<whatyourswascalled>" renamed to "lhr".
```

## Prepare the London Cluster for Coherence

We will create a new Kubernetes namespace and then install the [Coherence Operator](https://github.com/oracle/coherence-operator) into our London cluster. 

Create  new namespace to isolate the resources that are part of our lab, in the cloud shell terminal issue the command:

```
$ kubectl create namespace coherence-demo-ns
```

Then add the Coherence Operator helm chart repo to the local, pre-installed helm utility. 

```
$ helm repo add coherence https://oracle.github.io/coherence-operator/charts
```

Then update the local repos:

```
helm repo update
```

Install the operator into our London cluster:

```
helm install coherence-operator --namespace coherence-demo-ns coherence/coherence-operator
```

Check that the operator pod is running:

```
kubectl get pods --namespace coherence-demo-ns
```

The pod should show as being in the Running state. The operator is now ready to intercept the creation of new, custom coherence cluster resources being sent to the Kubernetes API. 

## Deploy the primary Coherence cluster 

We will deploy the first Coherence cluster in the London OKE cluster. A pre-built manifest file defines all the resources needed to run a Coherence cluster for object storage and another, storage-disabled Coherence cluster for the user interface. The manifests are stored in a git repo that can be cloned in Cloud Shell and then deployed. Clone the repo:

```
$ git clone https://gitlab.osc-cloud.com/coherence-workshop/coherence-demo.git
```

The contents of the repo will be copied to your Cloud Shell environment. Once complete change into the new directory:

```
$ cd coherence-demo/
```

Review the manifest for the primary cluster:

```
$ less ~/coherence-demo/yaml/primary-cluster.yaml
```

The manifest defines two Coherence clusters, one for storage and one for the user interface. Note that the Kind of the resource is "Coherence". This is a CRD or custom resource definition that is intercepted by the Coherence Operator previously deployed, the operator will take care of creating all the required Kubernetes resources; pods, statefulsets, services, persistent volumes, etc. Note that it covers aspects familiar to a Coherence user such as JVM args and cache config file.

NOTE: It is possible to Maximise the Cloud Shell window in order to make looking at the file a little easier.

```yaml
apiVersion: coherence.oracle.com/v1
kind: Coherence
metadata:
  name: primary-cluster-storage
```

The manifest also contains details of the container image used as well as the number of replicas used:

```yaml
image: lhr.ocir.io/oscemea001/coherence/coherence-demo:4.0.0-SNAPSHOT
  imagePullPolicy: Always
  replicas: 3
```

Press "q" to exit viewing the manifest.

Deploy the primary Coherence cluster to the London OKE cluster:

```
$ kubectl apply -f -n coherence-demo-ns ~/coherence-demo/yaml/primary-cluster.yaml
```

To check the state of the Coherence cluster issue the command:

```
$ kubectl get coherence -n coherence-demo-ns
NAME                      CLUSTER           ROLE      REPLICAS   READY   PHASE
primary-cluster-http      primary-cluster   http      1          1       Ready
primary-cluster-storage   primary-cluster   storage   2          2       Ready
```

The basic kubernetes resources created by the Coherence operator can be seen with the following command which will list the pods, services, deployments, replicasets and statefulsets in the coherence-demo-ns namespace:

```
$ kubectl get all -n coherence-demo-ns
NAME                                      READY   STATUS    RESTARTS   AGE
pod/coherence-operator-845c87bd5f-766px   1/1     Running   0          18h
pod/primary-cluster-http-0                1/1     Running   0          4h46m
pod/primary-cluster-http-1                1/1     Running   1          20h
pod/primary-cluster-storage-0             1/1     Running   0          4h46m
pod/primary-cluster-storage-1             1/1     Running   2          20h
pod/primary-cluster-storage-2             1/1     Running   0          4h46m

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/coherence-operator-metrics           ClusterIP   10.96.220.115   <none>        8383/TCP,8686/TCP   5d1h
service/primary-cluster-http-http            NodePort    10.96.112.56    <none>        8080:32636/TCP      20h
service/primary-cluster-http-metrics         ClusterIP   10.96.171.136   <none>        9612/TCP            20h
service/primary-cluster-http-sts             ClusterIP   None            <none>        7/TCP               20h
service/primary-cluster-http-wka             ClusterIP   None            <none>        7/TCP               20h
service/primary-cluster-storage-federation   ClusterIP   10.96.107.5     <none>        40000/TCP           20h
service/primary-cluster-storage-metrics      ClusterIP   10.96.80.100    <none>        9612/TCP            20h
service/primary-cluster-storage-np           NodePort    10.96.149.115   <none>        40000:31602/TCP     4d19h
service/primary-cluster-storage-sts          ClusterIP   None            <none>        7/TCP               20h
service/primary-cluster-storage-wka          ClusterIP   None            <none>        7/TCP               20h

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coherence-operator   1/1     1            1           5d1h

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/coherence-operator-845c87bd5f   1         1         1       5d1h

NAME                                       READY   AGE
statefulset.apps/primary-cluster-http      2/2     20h
statefulset.apps/primary-cluster-storage   3/3     20h
```

## Connect to the Demo App on the London OKE Cluster

The user interface of the Coherence Demo app is exposed as a NodePort service in Kubernetes, i.e. a port is exposed on the public IP address of every kubernetes worker node. Note that in a production scenario other choices are available for exposing the UI that might be more suitable! 

List the public IP addresses of the worker nodes:

```
$ kubectl get nodes -owide
```

The output will list the addresses under EXTERNAL-IP column, copy any one and enter the URL http://<EXTERNAL-IP>:32636/application/index.html. This will open the UI:

![image-20200929152901818](image-20200929152901818.png)

## Scale the OKE Cluster

The OKE cluster has been provisioned with two worker nodes, we will add a third worker node by scaling the cluster's only node pool. We will then scale our Coherence cluster to use the new compute capacity. 

Ensure you are still logged in to the OCI Console and on the London cluster's homepage. If not access the hamburger in the top left hand corner then select Developer Services then select Kubernetes Clusters and pick your assigned cluster form the list. In the resources menu on the left select Node Pools

 ![Screenshot from 2020-09-29 16-16-16](Screenshot%20from%202020-09-29%2016-16-16.png)

On the subsequently displayed node pool click on pool1 to see the pool details.

![Screenshot from 2020-09-29 16-22-47](Screenshot%20from%202020-09-29%2016-22-47.png)

Click scale to change the number of worker nodes from 2 to 3. This will cause a new virtual machine to be provisioned, the kubernetes software to be installed and joined to the OKE cluster. 

![image-20200929162615227](image-20200929162615227.png)

Change number of nodes to 3 and press the blue Scale button. 

Back in the Cloud Shell issue the following command to see the new node being added:

```
kubectl get nodes -w
```

The -w flag will cause the command to wait until the new node is added. This usually takes a few moments so this is a good point to have a tea break! 

Once the new node is added stop the command with ^c. Check  you have three nodes in the ready state with:

```
kubectl get nodes
```

## Scale the Coherence Cluster

Now that we've added more compute capacity to the Kubernetes cluster we will scale our Coherence cluster. 

First check how the existing two pods of the storage cluster are distributed across the OKE worker nodes:

```
$ kubectl get po -n coherence-demo-ns -o wide -l coherenceRole=storage
```

This will show that the two pods are running on different worker nodes, the Coherence operator enforces anti-affinity. 

Now increase the replica count of the coherence storage cluster from 2 to 3. There are many ways to do this but we will edit the manifest file and reapply it. Open the primary-cluster.yaml in vi :

```
$ vi ~/coherence-demo/yaml/primary-cluster.yaml
```

Use the arrow keys to move the cursor to the replica field value of the primary-cluster-storage-resource:

```yaml
coherence:
    cacheConfig: cache-config.xml
    metrics:
      enabled: true
      port: 9612
  image: lhr.ocir.io/oscemea001/coherence/coherence-demo:4.0.0-SNAPSHOT
  imagePullPolicy: Always
  replicas: 2
---
```

With the 2 selected press the r key and then the 3 key to replace the value. Press escape then wq to write and quit vi. 

Now apply the manifests again:

```
$ kubectl apply -f ~/coherence-demo/yaml/primary-cluster.yaml -n coherence-demo-ns
```

Immediately check for the new node being added:

```
$ kubectl get po -n coherence-demo-ns -o wide -l coherenceRole=storage -w
```

You should see the new pod being added and transitioning to the Running state. Note the the pod is running on the new worker node.

In the application UI note that a new Coherence server is shown. The cache data will have been re-partitioned across all three servers:

![image-20200929173052997](image-20200929173052997.png) 

The primary OKE cluster has been built so that it's worker nodes lie across all three [availability domains](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) of the London region. Availability domains are isolated from each other, fault tolerant, and very unlikely to fail simultaneously. Because availability domains do not share infrastructure such as power or cooling, or the internal availability domain network, a failure at one availability domain within a region is unlikely to impact the availability of the others within the same region. Each availability domain has three [fault domains](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm#fault). A fault domain is a grouping of hardware and infrastructure within an availability domain. Additional OKE worker nodes will be distributed across fault domains within each region to provide additional levels of redundancy. To see how the OKE worker nodes are distributed across the OCI infrastructure issue the following command:

```
$ kubectl get nodes -o wide -L failure-domain.beta.kubernetes.io/zone,oci.oraclecloud.com/fault-domain
```

This will print the availability domain and fault domain of each worker node.

## Recovery from Pod Failure

We will now simulate a failure of a pod and observe Kubernetes automatic recovery. A pod in Kubernetes is the smallest depoyable unit and consists of one or more running containers. In our example the pods are managed by Kubernetes statefulsets which control an ordered set of pods each with a stable network identity, ideal for applications like Coherence. The Kubernetes controller maintains the number of pods in the statefulset at the number specified when deployed, the "desired state".

In the Cloud Shell list the pods that constitute the storage cluster:

```
$ kubectl get po -n coherence-demo-ns -o wide -l coherenceRole=storage
NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
primary-cluster-storage-0   1/1     Running   1          16h     10.244.0.168   10.0.10.2   <none>           <none>
primary-cluster-storage-1   1/1     Running   0          2m59s   10.244.1.40    10.0.10.4   <none>           <none>
primary-cluster-storage-2   1/1     Running   1          15h     10.244.0.3     10.0.10.6   <none>           <none>
```

This shows that each pod is named "primary-cluster-storage-" followed by an ordinal, e.g. primary-cluster-storage-0. We will delete one of these pods. You will also be able to view the recovery from the demo applications UI so try and arrange your browser windows to show the UI alongside the Cloud Shell. To delete the pod issue the command:

```
$ kubectl delete po primary-cluster-storage-0 -n coherence-demo-ns
```

Immediately once this completes issue the command 

```
$ kubectl get po -n coherence-demo-ns -o wide -l coherenceRole=storage -w
```

to view the pod being restarted by the Kubernetes controller. In many cases the restart will be very quick and you may see that the primary-cluster-storage-0 pod is already running but will show an AGE of a few seconds. The application's UI should show the number of coherence servers drop to two, the data being re-partitioned, then a third coherence server reappearing and then the data being re-partitioned again. 

## Recovery from Node Failure

We will now remove a worker node from the Kubernetes cluster and observe how Kubernetes maintains our Coherence cluster at three servers. Locate the London OKE cluster homepage in the OCI console. 

![Screenshot from 2020-09-29 16-16-16](Screenshot%20from%202020-09-29%2016-16-16-1601451821972.png)

Click Node Pools to view the single node pool:

![Screenshot from 2020-09-29 16-22-47](Screenshot%20from%202020-09-29%2016-22-47-1601451877786.png)

Select Scale and on the resulting screen change 3 nodes to 2 and press the blue Scale button. The last worker node to be added will be removed. Monitor the state of the application via the application's UI and also by running:

```
$ kubectl get po -n coherence-demo-ns -o wide -l coherenceRole=storage -w
```

You should see that the number of pods is retsored to 3 and the application rebalances the cache data as the number of cache servers drops and is then restored to the desired state. 

## Cache Federation Across OCI Regions

