# Setup Openshift CRC instance on Google Cloud

## Reference

Visit https://crc.dev/crc/getting_started/getting_started/introducing/


## 1. Set GCP Project

```bash
PROJECT_ID=devops-lab01

gcloud config set project $PROJECT_ID

```
## 2. Create Custom Subnet and Firewall Rule

```bash
VPC_NAME=vpc-crc
SUBNET_1=vpc-crc-subnet01
REGION=us-central1
FW_RULE1=ssh
FW_RULE2=https

gcloud compute networks create $VPC_NAME --subnet-mode=custom

gcloud compute networks subnets create $SUBNET_1 \
   --network $VPC_NAME \
   --region $REGION \
   --range 10.0.0.0/16

gcloud compute firewall-rules create $FW_RULE1 \
	--network=$VPC_NAME \
	--action=ALLOW \
	--rules=tcp:22 \
	--source-ranges=0.0.0.0/0

gcloud compute firewall-rules create $FW_RULE2 \
	--network=$VPC_NAME \
	--action=ALLOW \
	--rules=tcp:443 \
	--source-ranges=0.0.0.0/0

```

## 3. Enable Nested Virtualization

Only when using Google Organizations.

```bash

gcloud resource-manager org-policies \
  disable-enforce compute.disableNestedVirtualization \
  --project=$PROJECT_ID

```

## 4. Create a VM

```bash
VM_NAME="openshift-crc"
VM_ZONE="us-central1-c"

gcloud compute instances create $VM_NAME \
    --machine-type=custom-16-32768 \
    --enable-nested-virtualization \
    --min-cpu-platform="Intel Haswell" \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --zone $VM_ZONE \
    --subnet=$SUBNET_1 \
    --boot-disk-size 300G --boot-disk-type pd-standard \
    --image-project=ubuntu-os-cloud --image-family=ubuntu-2204-lts \
    --metadata=enable-oslogin=TRUE

```

without OS login,

```bash
ssh-keygen -t ed25519 -N "" -f "id_rsa_crc"
sed "s/ssh-ed25519/$(whoami):ssh-ed25519/" "id_rsa_crc.pub" > ssh-metadata
gcloud compute instances add-metadata $VM_NAME \
    --zone $ZONE --metadata-from-file ssh-keys=ssh-metadata
gcloud compute ssh $VM_NAME --ssh-key-file="id_rsa_crc" --tunnel-through-iap
```

## 5. Install CRC

Login to Vm and Install Openshift CRC.

```bash
sudo apt-get update
sudo apt install -y qemu-kvm libvirt-daemon libvirt-daemon-system network-manager
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
tar -xvf crc-linux-amd64.tar.xz
sudo mv crc-linux-*/crc /usr/local/bin/
crc config set cpus 12
crc config set memory 30517
crc config set disk-size 200
crc config set network-mode user
crc config set consent-telemetry no
crc config view
crc cleanup
crc setup
exit
```

## 6. Reset the VM

```bash
gcloud compute instances reset $VM_NAME --zone $VM_ZONE
gcloud compute ssh $VM_NAME --zone $VM_ZONE
```

## 7. Start the CRC server

You have to prepare a `pull secret` on https://cloud.redhat.com/openshift/create/local in advance.

```bash
crc setup
crc daemon &
crc start
```

Openshift CRC started and display the below messages
```bash
Started the OpenShift cluster.

The server is accessible via web console at:
  https://console-openshift-console.apps-crc.testing

Log in as administrator:
  Username: kubeadmin
  Password: 5HCov-kARJo-LTR7q-uSQSC

Log in as user:
  Username: developer
  Password: developer

Use the 'oc' command line interface:
  $ eval $(crc oc-env)
  $ oc login -u developer https://api.crc.testing:6443
```

## 8. Connect to the server

in Windows, open host file in C:\Windows\Systems32\drivers\etc and add the following lines.
Note: 
34.28.146.196 is Openshift CRC VM public IP address.

```text
34.28.146.196	console-openshift-console.apps-crc.testing 
34.28.146.196	canary-openshift-ingress-canary.apps-crc.testing 
34.28.146.196	console-openshift-console.apps-crc.testing
34.28.146.196	default-route-openshift-image-registry.apps-crc.testing 
34.28.146.196	downloads-openshift-console.apps-crc.testing 
34.28.146.196	oauth-openshift.apps-crc.testing
34.28.146.196	api.crc.testing
```

Open openshift GUI via browser:

https://console-openshift-console.apps-crc.testing

Login as an administrator, then request the resources.

```bash
eval $(crc oc-env)
oc login -u kubeadmin https://api.crc.testing:6443
oc get co
oc get po --all-namespaces | wc -l
```

Try kubectl as well.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
export KUBECONFIG=$HOME/.crc/machines/crc/kubeconfig
kubectl get po --all-namespaces | wc -l
```

## 9. Make a StorageClass

@see https://github.com/code-ready/crc/wiki/Dynamic-volume-provisioning

## 10. HAProxy Setup
HA Proxy is optional when openshift CRC installed at on-prmise VM.

Make sure you have haproxy and a few other things

```bash
sudo dnf -y install haproxy policycoreutils-python-utils jq
```

Modify the firewall on the server

```bash
sudo systemctl start firewalld
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo systemctl restart firewalld
sudo semanage port -a -t http_port_t -p tcp 6443

```

Configure haproxy on the server
The steps below will create an haproxy config file with placeholders, update the SERVER_IP and CRC_IP using sed, and copy the new file to the correct location. If you would like to edit the file manually, feel free :)

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
tee haproxy.cfg &>/dev/null <<EOF
global
        debug

defaults
        log global
        mode    http
        timeout connect 5000
        timeout client 5000
        timeout server 5000

frontend apps
    bind SERVER_IP:80
    bind SERVER_IP:443
    option tcplog
    mode tcp
    default_backend apps

backend apps
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server webserver1 CRC_IP check

frontend api
    bind SERVER_IP:6443
    option tcplog
    mode tcp
    default_backend api

backend api
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server webserver1 CRC_IP:6443 check
EOF

export SERVER_IP=$(hostname --ip-address)
export CRC_IP=$(crc ip)
sed -i "s/SERVER_IP/$SERVER_IP/g" haproxy.cfg
sed -i "s/CRC_IP/$CRC_IP/g" haproxy.cfg
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg
sudo systemctl start haproxy

```

Setup NetworkManager on the client machine
NetworkManager needs to be configured to use dnsmasq for DNS. Make sure you have dnsmasq installed:

```bash
sudo dnf install dnsmasq
```

Add a file to /etc/NetworkManager/conf.d to enable use of dnsmasq. (Some systems may already have this setting in an existing file, depending on what's been done in the past. If that's the case, continue on without creating a new file)

```bash
sudo tee /etc/NetworkManager/conf.d/use-dnsmasq.conf &>/dev/null <<EOF
[main]
dns=dnsmasq
EOF
Add dns entries for crc:

tee external-crc.conf &>/dev/nu
```
