# VKICKS Infra

This repo describes setup and maintenance of the hosting platform behind Skjutsgruppen.

# TODO

- [ ] Move the repo to the correct GitHub organisation
- [ ] Rename the repo to vkicks-infra
- [ ] Setup firewall rules
- [ ] Setup longhorn backups
- [ ] Create an SSH key and save in vault
- [ ] Setup automated build + deploy
- [ ] Create LICENSE

## Contributing

### Dependencies

You will need a bunch of CLI tools to be able to work with the infrastructure.

```shell
brew install jq kubens kubectl kubectx skaffold
```

### Access

Once you have kubectl installed, add the skjutsgruppen cluster context to your `~/.kube/config`. Ask someone in the project to give it to you.

If you need SSH access to the servers, you need to ask someone in the project to add your SSH key to each server.

## Cluster setup

The cluster is currently running on two instances. 2 vCPU, 4 GB RAM and 40 GB disk each. One node acts as the kubernetes master and the rest are workers.

### Server setup

After setting up a new server, SSH into it and apply the following configuration.

```shell
apt update
apt -y upgrade
apy -y insatll open-iscsi
systemctl enable --now iscsid

adduser skjutsgruppen --disabled-password
usermod -a -G sudo skjutsgruppen
visudo # Change the %sudo line to this:
# %sudo   ALL=(ALL) NOPASSWD:ALL

su - skjutsgruppen
mkdir .ssh
touch .ssh/authorized_keys
# Add SSH keys to .ssh/authorized_keys


```

### Kubernetes cluster

#### Setup master node

```shell
# SSH into the master node.

# Install K3s.
curl -sfL https://get.k3s.io | sh -

# Remove traefik
sudo kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
sudo kubectl -n kube-system delete helmcharts.helm.cattle.io traefik-crd

# Disable traefik from being deployed again.
sudo service k3s stop
sudo mv /var/lib/rancher/k3s/server/manifests/traefik.yaml /var/lib/rancher/k3s/server/manifests/traefik.yaml.skip

# Start K3s.
sudo service k3s start
```

Verify that you cluster master node is Ready by running `sudo kubectl get node`.

The next step is to get the kube config so you can setup the context on your local machine. Run `sudo cat /etc/rancher/k3s/k3s.yaml` and copy the necessary contents to your `~/.kube/config`.

#### Setup a worker node

Each worker node will join the cluster by running k3s in agent-mode. In order to join a node to a cluster, you need a _node-token_ from the master node.

```shell
# The follow is to be run on the master node.

# Get the node-token.
cat /var/lib/rancher/k3s/server/node-token
```

Once you have the node-token, SSH into the worker node and run the following.

```shell
# The following is to be run on the worker node you are setting up.

# Set master node IP.
# Replace 13.37.0.1 with the IP of the master node.
K3S_URL=https://13.37.0.1:6443

# Set the node-token.
# Replace NODE-TOKEN with the token you got from the master node.
K3S_TOKEN="NODE-TOKEN"

# Install
curl -sfL https://get.k3s.io | sh -

# Enable the k3s-agent
sudo systemctl enable --now k3s-agent
```

#### Install dependencies

We are running a few cluster-wide dependencies that are in charge of different functionality. These are installed by running `skaffold run` in this repository.

- **cert-manager** provides easy SSL certificate management with Letsencrypt
- **ingress-nginx** terminates HTTPS traffic and routes incoming requests to the correct service
- **longhorn** provides a cluster-wide storage

Once longhorn is installed you will have two default storage classes and you want to disable the one called _local-path_.

```shell
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

##### Longhorn backup

```shell
kubectl create secret generic backup-secret \
  --from-literal=AWS_ACCESS_KEY_ID="<key-id>" \
  --from-literal=AWS_SECRET_ACCESS_KEY="<access-key>" \
  --from-literal=AWS_ENDPOINTS=https://objects.dc-fbg1.glesys.net
```

Configure longhorn to use `s3://bucket-name@objects/` replacing _bucket-name_ with the name of your bucket and `backup-secret`.
