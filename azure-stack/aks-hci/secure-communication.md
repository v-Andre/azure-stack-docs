---
title: Secure communication between control plane nodes for AKS on Azure Stack HCI
description: Learn how to secure communication between in-cluster components.
author: aabhathipsay
ms.topic: how-to
ms.date: 03/04/2021
ms.author: aathipsa
ms.reviewer: 
---

# Secure communication with certificates  

Certificates are a means to build secure communication between in-cluster components. In AKS on Azure Stack HCI, zero-touch, out-of-the-box provisioning and management of certificates is provided for the Kubernetes built-in components. 

> [!NOTE]
> This preview release enables secure communication between in-cluster components by replacing self-signed certificates with certificates issued by a Certificate Authority (CA) for new target clusters. 

## Certificates and CAs

AKS on Azure Stack HCI generates and uses the following certificates and CAs: 

**Cluster CA**:
  - The API server has a Cluster CA, which signs certificates for one-way communication from the API server to `kubelet`.
  - Each `kubelet` also creates a Certificate Signing Request (CSR), which is signed by the Cluster CA, for communication from the kubelet to the API server.
  - The etcd key value store has a certificate signed by the Cluster CA for communication from etcd to the API server. 

**etcd CA**:
- The etcd key value store has an etcd CA that signs certificates to authenticate and authorize data replication between etcd replicas in the cluster.

**Front Proxy CA**
- The Front Proxy CA secures communication between the API server and the extension API server.

### Certificate provisioning 

Certificate provisioning for kubelets is done using [TLS bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/). For all other certificates, YAML-based key and certificate creation is used. 

- The certificates are stored at /etc/kubernetes/pki
- The keys are RSA 4096, EcdsaCurve: P384 

> [!NOTE]
> The root certificates are valid for 10 years. All other non-root certificates are short-lived and valid for four days.

### Certificate renewal and management

Non-root certificates are automatically renewed. Note that support for certificate revocation is not available in this preview release.

All control plane certificates for Kubernetes are managed except for the following certificates:

- Kubelet server certificate 
- Kubeconfig client certificate 

We recommend using [Active Directory single sign-in](./ad-sso.md) for user authentication as a security best practice.

## Enable this capability for your cluster

Use the `-enableCertificateRotation` parameter of the [New-AksHciCluster](./new-akshcicluster.md) command to enable secure communication and automate certificate management as shown below: 

```powershell
New-AksHciCluster -name mynewcluster -enableCertificateRotation 
```

Once you deploy a new cluster with the `-enableCertificateRotation` parameter, you cannot disable this feature. 

## Troubleshoot and maintenance

Refer to the following scripts and documentation for logging and monitoring:

- [Logging](https://github.com/microsoft/AKS-HCI-Apps/tree/main/Logging)
- [Monitoring](https://github.com/microsoft/AKS-HCI-Apps/tree/main/Monitoring#certs-and-keys-monitoring)

### Renew certificates for worker nodes

For worker nodes, failed certificate renewals are logged by the *certificate-renewal-worker* pod. If the certificate renewal continues to fail on a worker node and the certificates expire, the node will be removed and a new worker node created in its place. 

Here's an example of viewing the logs for the pod with the prefix *certificate-renewal-worker*: 

```powershell
kubectl.exe --kubeconfig .\testcluster-kubeconfig -n=kube-system get pods 
```

```Output
NAME                                           READY   STATUS             RESTARTS   AGE 
… 
certificate-renewal-worker-6f68k               1/1     Running            0          6d 
```

To get the logs from the certificate renewal pod:

```powershell
kubectl.exe --kubeconfig .\testcluster-kubeconfig -n=kube-system logs certificate-renewal-worker-6f68k
```

### Renew certificates for control plane nodes

For control plane nodes, failed certificate renewals are logged by the certificate-renewal-controller pod. If certificates expire on a control plane node, it may eventually become unreachable by other nodes. If all control plane nodes enter this state, the cluster will become inoperable due to a TLS failure. You can confirm whether the cluster has entered this state by trying to access it using `kubectl`, and then verifying that the connection has failed if there's an error message related to expired x509 certificates. 

Here's an example of viewing the logs for the pod with the prefix *certificate-renewal-controller*:
```powershell
kubectl.exe --kubeconfig .\testcluster-kubeconfig -n=kube-system get pods 
```

```Output
NAME                                           READY   STATUS             RESTARTS   AGE 
… 
certificate-renewal-controller-2cdmz               1/1     Running            0          6d 
```

To get the logs from the certificate renewal pod:

```powershell
kubectl.exe --kubeconfig .\testcluster-kubeconfig -n=kube-system logs certificate-renewal-controller-2cdmz
```

Control plane nodes can’t be recreated like worker nodes, but you can use the **Repair-AksHciClusterCerts** module to help fix errors related to expired certificates. If the cluster begins to fail due to expired certificates, run the command as below: 

```powershell
Repair-AksHciClusterCerts -Name mytargetcluster 
```

If the cluster becomes unreachable via `kubectl`, you can find the logs in the `/var/log/pods` folder.

## Next steps

In this how-to guide, you learned how to enable the encryption of etcd secrets for new workload clusters. Next, you can:
- [Deploy a Linux applications on a Kubernetes cluster](./deploy-linux-application.md).
- [Deploy a Windows Server application on a Kubernetes cluster](./deploy-windows-application.md).