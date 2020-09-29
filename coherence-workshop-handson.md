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

We will now configure kubectl in cloud shell to work with the London OKE cluster. Click on the "hamburger" icon in the top left to open the services menu and locate the Developer Services entry and then the Kubernetes Clusters entry.

![image-20200929084456958](image-20200929084456958.png)

On the OKE clusters homepage locate the cluster assigned to you. Ensure you are in the correct compartment as specified in the student details by checking the compartment drop down in the left of the screen. Click your assigned OKE cluster to view it's details, then select the blue "Access Cluster" button. 

![image-20200929090516869](image-20200929090516869.png)

Configuring kubectl is a case of just following the instructions on the screen. First press the "Launch Cloud Shell" button, after a few moments a terminal will launch at the bottom of your browser window. Then copy the oci cli command and paste it into the cloud shell terminal. Your cloud shell is pre-authenticated against your OCI account so no credentials are needed. The command will copy the kube config file to the standard location at ~/.kube/config. You should now be able to query your OKE cluster by running using the pre-installed kubectl command:

```bash
kubectl get nodes -o wide
```

in the cloud shell window.

![image-20200929091157353](image-20200929091157353.png)

We will now rename the context created to allow us to switch between the London and Frankfurt clusters later in the lab. Query the current context with the command:

```
$ kubectl config get-contexts
CURRENT   NAME                  CLUSTER               AUTHINFO           NAMESPACE
*         context-cqtsyzvge2w   cluster-cqtsyzvge2w   user-cqtsyzvge2w  
```

Rename the context to lhr which should prove to be a little more memorable! 

```
$ kubectl config rename-context context-cqtsyzvge2w lhr
Context "context-cqtsyzvge2w" renamed to "lhr".
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
$ less yaml/primary-cluster.yaml
```

The manifest defines two Coherence clusters, one for storage and one for the user interface. Note that the Kind of the resource is "Coherence". This is a CRD or custom resource definition that is intercepted by the Coherence Operator previously deployed, the operator will tale care of creating all the required Kubernetes resources; pods, statefulsets, services, persistent volumes, etc. Note that it covers aspects familiar to a Coherence user such as JVM args and cache config file.

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

Deploy the primary Coherence cluster to the London OKE cluster:

```
$ kubectl apply -f yaml/primary-cluster.yaml
```

To check the state of the Coherence cluster issue the command:

```
$ kubectl get coherence -n coherence-demo-ns
NAME                      CLUSTER           ROLE      REPLICAS   READY   PHASE
primary-cluster-http      primary-cluster   http      2          2       Ready
primary-cluster-storage   primary-cluster   storage   2          2       Ready
```

The basic kubernetes resources created by the operator can be seen with the following command which will list the pods, services, deployments, replicasets and statefulsets in the coherence-demo-ns namespace:

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

