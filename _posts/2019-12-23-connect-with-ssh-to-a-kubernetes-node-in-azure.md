---
layout: post
title: Connect with SSH to a Kubernetes node in Azure
tags: [azure, kubernetes]
---

A couple of how-tos and tutorials have been published on the internet about how to connect with SSH to a Kubernetes cluster node in Azure, but most of them seems to be outdated and didn't work for me. I'm mostly following the [official](https://docs.microsoft.com/en-us/azure/aks/ssh) solution, recently posted by Microsoft, but I couldn't change the public SSH key on the cluster with their method, so I share my workaround for scale set-based Linux clusters. The advantage of this solution is that you don't have to set a public IP adress to the nodes, so it is more secure.

You have to install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) first, log in with `az login` and determine the name of your Kubernetes cluster and in which resource group your cluster is in.

```bash
CLUSTER_RESOURCE_GROUP=$(az aks show --resource-group yourResourceGroup --name yourKubernetesCluster --query nodeResourceGroup -o tsv)
SCALE_SET_NAME=$(az vmss list --resource-group $CLUSTER_RESOURCE_GROUP --query [0].name -o tsv)
```

The above commands gets the resource group and scale set of the Azure Kubernetes Service (AKS) cluster and stores them in the variables.
To connect to the cluster you have to generate an SSH key, I suggest you to use `ssh-keygen`. Further information about the key generation can be found [here](https://www.ssh.com/ssh/keygen).

```bash
az vmss update \
    --resource-group $CLUSTER_RESOURCE_GROUP \
    --name $SCALE_SET_NAME \
    --add virtualMachineProfile.osProfile.linuxConfiguration.ssh.publicKeys '{"path": "/home/azureuser/.ssh/authorized_keys", "keyData": "yourPublicKey"}'

az vmss update \
    --resource-group $CLUSTER_RESOURCE_GROUP \
    --name $SCALE_SET_NAME \
    --remove virtualMachineProfile.osProfile.linuxConfiguration.ssh.publicKeys 0
```

The first line of the above example set the RSA public key of the cluster and to do so, you have to replace the value of the `keyData` property with your own public key. The second command deletes the old public key from the cluster.

{: .no-break}
```bash
$ kubectl get nodes -o wide

NAME                                STATUS     ROLES   AGE     VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-agentpool-xxxxxxxx-vmss000000   NotReady   agent   6d22h   v1.13.12   10.240.0.4    <none>        Ubuntu 16.04.6 LTS   4.15.0-1064-azure   docker://3.0.7
aks-agentpool-xxxxxxxx-vmss000001   NotReady   agent   5d3h    v1.13.12   10.240.0.5    <none>        Ubuntu 16.04.6 LTS   4.15.0-1064-azure   docker://3.0.7
```

You need the internal IP address of the node which you would like to connect with SSH, the above command can help you to get that. If you don't have the Kubernetes command-line tool, `kubectl`, installed on your computer, [this](https://kubernetes.io/docs/tasks/tools/install-kubectl/) guide can help you how to install and set up that.

```bash
kubectl run --generator=run-pod/v1 -it --rm aks-ssh --image=debian
```

To create an SSH connection to an AKS node, you run a helper pod in your AKS cluster. This helper pod provides you with SSH access into the cluster and then additional SSH node access. The above command runs a debian container image and attach a terminal session to it. This container can be used to create an SSH session with any node in the AKS cluster.

```bash
apt-get update && apt-get install openssh-client -y
```

Once the terminal session is created, install an SSH client with `apt-get`, which you will use to connect to the AKS node.

```bash
kubectl cp ~/.ssh/id_rsa $(kubectl get pod -l run=aks-ssh -o jsonpath='{.items[0].metadata.name}'):/id_rsa
```

Open a new terminal window, which is not connected to the helper pod and use the above command to copy the previously created SSH private key to the helper pod. You may need to change the name or location of the key (*~/.ssh/id_rsa*).

```bash
chmod 0600 id_rsa
```

Once the upload is successful, change the terminal to the helper pod session and update the permission of the copied file with the above command.

```bash
$ ssh -i id_rsa azureuser@10.240.0.4

ECDSA key fingerprint is SHA256:A6rnRkfpG21TaZ8XmQCCgdi9G/MYIMc+gFAuY9RUY70.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.240.0.4' (ECDSA) to the list of known hosts.

Welcome to Ubuntu 16.04.5 LTS (GNU/Linux 4.15.0-1018-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    https://www.ubuntu.com/business/services/cloud

[...]

azureuser@aks-nodepool1-79590246-0:~$
```

Now you can connect to the chosen AKS node with the above command, the default username of the node is `azureuser`. When you are done, `exit` the SSH session and the helper pod will be automatically deleted.