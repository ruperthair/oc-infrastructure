# RecipeRadar Infrastructure

## Installation

This section documents the steps required to set up a fresh RecipeRadar environment.

### Configure host system

#### Install dependencies

```
# Keyrings
wget -qO - https://artifacts.opensearch.org/publickeys/opensearch.pgp | gpg --dearmor --batch --yes --output /etc/apt/keyrings/opensearch-keyring.gpg
wget -qO - https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.28/Debian_12/Release.key | gpg --dearmor --batch --yes --output /etc/apt/keyrings/kubic-libcontainers-crio.gpg
wget -qO - https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor --batch --yes --output /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Repository sources
echo 'deb [signed-by=/etc/apt/keyrings/opensearch-keyring.gpg] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main' | tee /etc/apt/sources.list.d/opensearch-2.x.list
echo 'deb [signed-by=/etc/apt/keyrings/kubic-libcontainers-crio.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.28/Debian_12/ /' | tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

# Dependency installation
apt update
apt install haproxy postgresql rabbitmq-server squid-openssl containers-storage
apt install cri-o  # repo: kubic
apt install opensearch  # repo: opensearch
apt install kubeadm  # repo: kubernetes
```

#### Enable crio container seccomp profile
```
vim /etc/crio/crio.conf
...
seccomp_profile = "/usr/share/containers/seccomp.json"
```

#### Configure container storage
```
vim /usr/share/containers/storage.conf
...
driver = "overlay"
...
rootless_storage_path = "/mnt/ephemeral/containers/user-storage/"
...
additionalimagestores = [
    "/mnt/ephemeral/containers/user-storage/"
]
```

#### Enable ipv4 packet forwarding
```
vim /etc/sysctl.d/99-sysctl.conf
...
net.ipv4.ip_forward=1
...
sysctl --system
```

#### Install required kernel modules
```
echo br_netfilter >> /etc/modules
```

#### Create a persistent dummy network interface
```
vim /etc/systemd/network/10-dummy0.netdev
...
[NetDev]
Name=dummy0
Kind=dummy
...
vim /etc/systemd/network/20-dummy0.network
...
[Match]
Name=dummy0

[Network]
Address=192.168.100.1/32
Address=fe80::0100:0001/128
DHCP=no
IPv6AcceptRA=no
...
vim /etc/systemd/network/30-dummy0.link
...
[Match]
OriginalName=dummy0

[Link]
MACAddressPolicy=random
...
systemctl restart systemd-networkd
```

#### If NetworkManager is installed, prevent the host from autoconfiguring Calico network interfaces

```
vim /etc/NetworkManager/conf.d/calico.conf
...
[keyfile]
unmanaged-devices=interface-name:cali*;
```

### Configure services

#### OpenSearch
```
vim /etc/opensearch/opensearch.yml
...
network.host: 192.168.100.1
...
cluster.initial_cluster_manager_nodes: ["192.168.100.1"]
...
plugins.security.disabled: true
script.painless.regex.enabled: false
...
path.repo: "/mnt/backup/opensearch"
```

#### PostgreSQL
```
vim /etc/postgresql/*/main/postgresql.conf
...
listen_addresses = '192.168.100.1,fe80::0100:0001%dummy0'
...
max_connections = 200
...
shared_buffers = 2GB
...
synchronous_commit = off
...
effective_cache_size = 2GB
```

```
vim /etc/postgresql/*/main/pg_hba.conf
...
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    api             api             fe80::0100:0001/128     trust
host    api             api             192.168.100.1/32        trust
host    api             api             172.16.0.0/12           trust
...
```

#### RabbitMQ
```
vim /etc/rabbitmq/rabbitmq.conf # new file?
...
loopback_users = none
...
vim /etc/rabbitmq/rabbitmq-env.conf
...
NODE_IP_ADDRESS=192.168.100.1
...
```

# run from inside a clone of this repo

#### Squid
```
vim /etc/squid/squid.conf
...
# http_port 3128

cache_dir aufs /mnt/persistence/squid 32768 16 256
...
cp etc/squid/conf.d/recipe-radar.conf /etc/squid/conf.d/recipe-radar.conf
/usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 512MB
sh -x etc/squid/create-certificates.sh
```

#### HAProxy
```
cp etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg
```

### Start system services
```
for service in systemd-networkd opensearch postgresql rabbitmq-server squid haproxy crio kubelet;
do
    systemctl enable ${service}.service
    systemctl restart ${service}.service
done
```

### Initialize the application database
```
sudo -u postgres createuser api
sudo -u postgres createdb api
```

### Register the OpenSearch snapshot repository
```
curl -XPUT -H 'Content-Type: application/json' 'http://192.168.100.1:9200/_snapshot/reciperadar' --data '{"type": "fs", "settings": {"location": "/mnt/backup/opensearch"}}'
```

### Configure kubernetes cluster

#### Initialize cluster
```
kubeadm init --apiserver-advertise-address=192.168.100.1 --pod-network-cidr=172.16.0.0/12
```

```
IMPORTANT: DROP YOUR PRIVILEGES AT THIS POINT

Everything from this point onwards can be performed as an unprivileged user
```

#### Configure kubectl user access
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Deploy kubernetes infrastructure components
```
for component in k8s/*;
do
	kubectl apply -f ${component}
done;
```

#### Remove scheduling constraint from host node
```
kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes $(hostname) node-role.kubernetes.io/master:NoSchedule-
```

#### Provide the proxy certificate to all cluster services
```
kubectl create secret generic proxy-cert --from-file=/etc/squid/certificates/ca.crt
```

#### Run smoke tests
```
# Make a request to a deployed service
curl -H 'Host: frontend' localhost:30080

# Make a request via the Kubernetes ingress controller
curl localhost
```

## Operations

### Upgrade Kubernetes Infrastructure

Updated versions of the Kubernetes toolset (`kubeadm`, `kubectl`, `kubelet`) are released on a fairly regular basis.

It is worth upgrading to stay current with the latest releases, but it's also important to test that the changes are safe before applying them to production; investigation and remediation of problems can take time.

Follow the upgrade process below in a **development** environment before applying the same steps in production.

```bash
# Plan an upgrade
$ sudo kubeadm upgrade plan

# Apply the upgrade
$ sudo kubeadm upgrade apply ...

# Wait, and then ensure that containers are running and in a good state
$ sleep 60
$ kubectl get pods

# Ensure that a non-critical service can deploy to the cluster
$ make build && make deploy
```

### Renew TLS certificates

TLS certificates ensure that users of the application receive the genuine, intended, unmodified software that we build, and that they can verify that they have received it from us.

Our TLS certificates are generated by [Let's Encrypt](https://letsencrypt.org/), an open certificate authority that can provide certificates at zero cost to anyone who owns an Internet domain name.

Certificates are short-lived and expire after three months, so we regenerate certificates towards the end of that period.

Renewing certificates requires a few steps:

1. Requesting a certificate from Let's Encrypt
1. Authenticating the certificate request by updating our DNS entries with an ACME challenge response record
1. Authenticating the certificate request by deploying an ACME challenge response file to our web server

The `certbot` tool provided by the [Electronic Frontier Foundation](https://www.eff.org/) provides support for these operations.  To renew certificates, walk through the following procedure:

```bash
$ certbot certonly --manual -d '*.reciperadar.com' -d 'reciperadar.com'
```

For verification purposes, you will need to update DNS records, and deploy the `frontend` application with an ACME challenge response:

```bash
$ echo "challenge-value" > static/.well-known/challenge-filename
$ make build && make deploy
```

Once verification is complete, `certbot` will write the certificate and corresponding private key to the `/etc/letsencrypt/live/reciperadar.com` directory.

The web server we use to handle TLS connections, `haproxy`, accepts a [`crt`](https://cbonte.github.io/haproxy-dconv/2.0/configuration.html#5.1-crt) configuration option containing the filename of a *combined* certificate and private key file.

In future it may be possible to configure `haproxy` to use separate certificate and key files as generated by `certbot` (see [haproxy/haproxy#785](https://github.com/haproxy/haproxy/issues/785)) -- or for `certbot` to emit combined certificate and private-key files (see [certbot/certbot#5087](https://github.com/certbot/certbot/issues/5087)); until then we can concatenate a certificate keyfile like so:

```bash
# Archive the existing certificate keyfile
$ mv /etc/ssl/private/reciperadar.com.pem /etc/ssl/private/reciperadar.com.pem.YYYYqQ

# Concatenate a generated certificate keyfile
$ test -e /etc/ssl/private/reciperadar.com.pem || cat /etc/letsencrypt/live/reciperadar.com/{fullchain,privkey}.pem > /etc/ssl/private/reciperadar.com.pem

# Restart the web server
$ systemctl restart haproxy
```

As a good practice, we should also revoke the archived and no-longer-used TLS certificate so that even if another party happened to obtain a copy of the certificate's private key, they would find it more difficult to impersonate the application during the remainder of the validity period.

```bash
$ certbot revoke --reason superseded --cert-path /etc/ssl/private/reciperadar.com.pem.YYYYqQ
```

### Regenerate proxy certificate

A self-signed `squid` proxy certificate is used so that outbound HTTPS (TLS) requests can perform verification against a certificate authority; albeit an internal one.  This does not provide security or integrity assurance - the proxy itself must perform verification on the requests that it makes.

The certificate is provided as a generic file-readable secret to containers in the Kubernetes cluster so that client libraries in any language can use it.

When the certificate expires, the following steps are required to regenerate and redistribute an updated certificate:

```sh
# Archive the existing certificates
$ ARCHIVE_DATE=$(date --reference /etc/squid/certificates/ca.crt '+%Y%m%d')
$ mkdir -p /etc/squid/certificates/archive/${ARCHIVE_DATE}/
$ mv /etc/squid/certificates/ca.{crt,key} /etc/squid/certificates/archive/${ARCHIVE_DATE}/

# Generate a new certificate signing key and certificate
$ openssl genrsa -out /etc/squid/certificates/ca.key 4096
$ openssl req -new -x509 -days 365 -key /etc/squid/certificates/ca.key -out /etc/squid/certificates/ca.crt

# Cleanup
$ unset ARCHIVE_DATE

# Refresh the certificate material in the Kubernetes cluster
$ kubectl delete secret generic proxy-cert
$ kubectl create secret generic proxy-cert --from-file=/etc/squid/certificates/ca.crt
```

You will also need to rebuild and deploy affected services so that their container images are updated with the latest certificate material.
