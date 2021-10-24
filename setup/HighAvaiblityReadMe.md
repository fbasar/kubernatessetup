Set up the first control plane node
Create a file called kubeadm-config.yaml with the following contents:
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
etcd:
    external:
        endpoints:
        - https://ETCD_0_IP:2379
        - https://ETCD_1_IP:2379
        - https://ETCD_2_IP:2379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
Note: The difference between stacked etcd and external etcd here is that the external etcd setup requires a configuration file with the etcd endpoints under the external object for etcd. In the case of the stacked etcd topology this is managed automatically.
Replace the following variables in the config template with the appropriate values for your cluster:
- `LOAD_BALANCER_DNS`
- `LOAD_BALANCER_PORT`
- `ETCD_0_IP`
- `ETCD_1_IP`
- `ETCD_2_IP`

```

The following steps are similar to the stacked etcd setup:

Run 
```
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

on this node.

Write the output join commands that are returned to a text file for later use.

Apply the CNI plugin of your choice. The given example is for Weave Net:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Steps for the rest of the control plane nodes
The steps are the same as for the stacked etcd setup:

Make sure the first control plane node is fully initialized.
Join each control plane node with the join command you saved to a text file. It's recommended to join the control plane nodes one at a time.
Don't forget that the decryption key from --certificate-key expires after two hours, by default.
Common tasks after bootstrapping control plane
Install workers
Worker nodes can be joined to the cluster with the command you stored previously as the output from the kubeadm init command:
```
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

Manual certificate distribution
If you choose to not use kubeadm init with the --upload-certs flag this means that you are going to have to manually copy the certificates from the primary control plane node to the joining control plane nodes.

There are many ways to do this. In the following example we are using ssh and scp:

SSH is required if you want to control all nodes from a single machine.

Enable ssh-agent on your main device that has access to all other nodes in the system:
```
eval $(ssh-agent)
```
Add your SSH identity to the session:
```
ssh-add ~/.ssh/path_to_private_key
```

SSH between nodes to check that the connection is working correctly.

When you SSH to any node, make sure to add the -A flag:
```
ssh -A 10.0.0.7
```

When using sudo on any node, make sure to preserve the environment so SSH forwarding works:

```
sudo -E -s
```

After configuring SSH on all the nodes you should run the following script on the first control plane node after running kubeadm init. This script will copy the certificates from the first control plane node to the other control plane nodes:

In the following example, replace CONTROL_PLANE_IPS with the IP addresses of the other control plane nodes.
```
USER=ubuntu # customizable
CONTROL_PLANE_IPS="10.0.0.7 10.0.0.8"
for host in ${CONTROL_PLANE_IPS}; do
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    # Quote this line if you are using external etcd
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
done
```
Caution: Copy only the certificates in the above list. kubeadm will take care of generating the rest of the certificates with the required SANs for the joining control-plane instances. If you copy all the certificates by mistake, the creation of additional nodes could fail due to a lack of required SANs.
Then on each joining control plane node you have to run the following script before running kubeadm join. This script will move the previously copied certificates from the home directory to /etc/kubernetes/pki:

```
USER=ubuntu # customizable
mkdir -p /etc/kubernetes/pki/etcd
mv /home/${USER}/ca.crt /etc/kubernetes/pki/
mv /home/${USER}/ca.key /etc/kubernetes/pki/
mv /home/${USER}/sa.pub /etc/kubernetes/pki/
mv /home/${USER}/sa.key /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
# Quote this line if you are using external etcd
mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
```
