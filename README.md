# kubeadm: Single Node Kubernetes Cluster on DigitalOcean

This tutorial will walk you through bootstrapping a single-node Kubernetes cluster on [Digital Ocean](https://www.digitalocean.com/) using [kubeadm](https://github.com/kubernetes/kubeadm).

## Tutorial

Create a single compute instance:

NOTE(jmodes): I was not able to get kubeadm running on a 512mb-sized machine.

```bash
doctl compute droplet create kubeadm-single-node-cluster \
  --image ubuntu-17-04-x64 \
  --region nyc1 \
  --size 1gb \
  --tag-names kubeadm-single-node-cluster,default-allow-outbound-all,default-allow-ssh \
  --enable-private-networking
```

Get information about your droplet:

```bash
DROPLET_ID=$(doctl compute droplet list --tag-name kubeadm-single-node-cluster -o json | jq '.[0].id' -r)
EXTERNAL_IP=$(doctl compute droplet get $DROPLET_ID -o json | jq '.[0].networks.v4 | .[] | select(.type == "public") | .ip_address' -r)
INTERNAL_IP=$(doctl compute droplet get $DROPLET_ID -o json | jq '.[0].networks.v4 | .[] | select(.type == "private") | .ip_address' -r)
```

Set up basic inbound and outbound firewall rules, including SSH access:

```bash
doctl compute firewall create \
  --name default-allow-outbound-all \
  --tag-names default-allow-outbound-all \
  --outbound-rules "protocol:icmp,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0"

doctl compute firewall create \
  --name default-allow-ssh \
  --tag-names default-allow-ssh \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0,address:::/0"
```

SSH onto your droplet using the password emailed to you, and reset your
password:

```bash
ssh root@${EXTERNAL_IP}
...
exit
```

Run the startup script:

```bash
cat startup.sh \
  | sed -e "s/\${EXTERNAL_IP}/${EXTERNAL_IP}/" -e "s/\${INTERNAL_IP}/${INTERNAL_IP}/" \
  | ssh root@${EXTERNAL_IP} 'cat > startup.sh && chmod +x startup.sh && ./startup.sh'
```

Enable secure remote access to the Kubernetes API server:

```bash
doctl compute firewall create \
  --name default-allow-kubeadm-single-node-cluster \
  --tag-names kubeadm-single-node-cluster \
  --inbound-rules protocol:tcp,ports:6443,address:0.0.0.0/0,address:::/0
```

> It may take a few minutes for the cluster to finish bootstrapping and the client config to become readable.

Fetch the client kubernetes configuration file:

```bash
scp root@${EXTERNAL_IP}:/etc/kubernetes/admin.conf \
  kubeadm-single-node-cluster.conf
```

Set the `KUBECONFIG` env var to point to the `kubeadm-single-node-cluster.conf` kubeconfig:

```bash
export KUBECONFIG=$(PWD)/kubeadm-single-node-cluster.conf
```

Set the `kubeadm-single-node-cluster` kubeconfig server address to the public IP address:

```bash
kubectl config set-cluster do-single-node-cluster \
  --kubeconfig kubeadm-single-node-cluster.conf \
  --server https://${EXTERNAL_IP}:6443
```


## Verification

List the Kubernetes nodes:

```bash
kubectl get nodes
```
```
NAME                          STATUS    AGE       VERSION
kubeadm-single-node-cluster   Ready     51s       v1.7.3
```

The node version reflects the `kubelet` version, therefore it might be different
than the `kubernetes-version` specified above.

Find out Kubernetes API server version:

```bash
kubectl version --short
```
```
Client Version: v1.7.4
Server Version: v1.8.0-beta.1
```

Create a nginx deployment:

```bash
kubectl run nginx --image nginx:1.13 --port 80
```

Expose the nginx deployment:

```bash
kubectl expose deployment nginx --type NodePort --port=80 --target-port=80
```

## Secure Droplet

If desired, remove SSH access:

```bash
doctl compute droplet untag $DROPLET_ID \
  --tag-name default-allow-ssh
```

[Or set up proper secure access via SSH keys.](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server#how-to-copy-a-public-key-to-your-server)

If desired, you can restrict your IP address to access the API server:

```bash
MY_IP=$(curl ifconfig.co)
doctl compute firewall update $(doctl compute firewall list -o json | jq '.[] | select(.name == "default-allow-kubeadm-single-node-cluster") | .id' -r) \
  --name default-allow-kubeadm-single-node-cluster \
  --tag-names kubeadm-single-node-cluster \
  --inbound-rules protocol:tcp,ports:6443,address:${MY_IP}/32
```

## Cleanup

```bash
doctl compute droplet delete kubeadm-single-node-cluster
```

```bash
doctl compute firewall delete $(doctl compute firewall list -o json | jq '.[] | select(.name == "default-allow-kubeadm-single-node-cluster") | .id' -r)
doctl compute firewall delete $(doctl compute firewall list -o json | jq '.[] | select(.name == "default-allow-outbound-all") | .id' -r)
doctl compute firewall delete $(doctl compute firewall list -o json | jq '.[] | select(.name == "default-allow-ssh") | .id' -r)
```

```bash
rm kubeadm-single-node-cluster.conf
```
