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

Configuring kubectl is a case of just following the instructions on the screen. First press the "Launch Cloud Shell" button, after a few moments a terminal will launch at the bottom of your browser window. Then copy the oci cli command and paste it into the cloud shell terminal. Your cloud shell is pre-authenticated against your OCI account so no credentials are needed. The command will copy the kube config file to the standard location at ~/.kube/config. You should now be able to query your OKE cluster by running 

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

Prepare the London Cluster for Coherence

