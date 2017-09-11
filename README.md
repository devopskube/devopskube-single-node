# Single Node CoreOS and Kubernetes Install

[CoreOS](https://coreos.com/) is the Host System we are using for the kubernetes installation. Since we are using CoreOS on a single-node (right now), in the following, you can find a description on how to setup a CoreOS single-node cluster. We are using a KVM-Base at [netcup](https://www.netcup.de/), but any other Hoster should work fine as well.

In the following, I am trying to re-play, what I did and provide some links and useful information on how to make this one work.

## Install CoreOS

Most of the following was taken from [The Hyperpessimist](https://xivilization.net/~marek/blog/2014/08/04/installing-coreos/). I chose the usual Ubuntu-Iso (note, use the ISO and do not install the image), which is offered by netcup as a base to be able to install coreos.

Please start the Rescue System on your netcup system and connect to it using SSH (User: root) and the provided key). No need to use the VNC-console anymore.

``
wget https://raw.github.com/coreos/init/master/bin/coreos-install
``

To install CoreOS in a "non-cloud" environment like netcup, you do need to provide an adopted cloud-config. This cloud-config file contains the public SSH-key, so that you are able to access the server using SSH with a key. Be sure to generate this key and put the public part of it into the `ssh-authorized-keys` section of the cloud-config file.  To be able to use this config-file, you do need to upload it to a server, where you can fetch this file via a wget, so that coreos can use it. If you don't know how to generate an SSH-key take a look into the [github help](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/). An example can be found in the repository [devopskube-single-node](https://www.github.com/devopskube/devopskube-single-node/).

In the [cloud-config](https://github.com/devopskube/devopskube-single-node/blob/master/cloud-config.yml) file you will furthermore find the public IP of the server, this needs to get adopted to your own needs as well. Right now it is defined as 192.168.0.1, which will not get routed to the internet. This file then needs to get copied to the instance, where you do like to install the coreos system. All those files can be copied to the system using SCP.

Afterwards the CoreOS installer can be called using the following commands:

``
chmod u+x coreos-install
bash coreos-install -d /dev/vda -C alpha -c cloud-config.yml
``

After the above described install, which can take some time, the system can be rebooted (please make sure, that the Rescue System is disabled). The system is then reachable via the configured IP and the configured SSH-key.

>NOTE: You will not be able to login to your system by password, just via the given SSH-key

## Install kubernetes

To install kubernetes on a single-node, I followed the [CoreOS - Single-Node Kubernetes Installation](https://coreos.com/kubernetes/docs/latest/kubernetes-on-vagrant-single.html). This installation description is mainly for vagrant, but since we do have a single-node install as well, this fits quite nicely.

### Prepare Installation

Before we do setup kubernetes, we do need some SSL certificates. This can be done using the scripts in the repository mentioned in the above description [CoreOS - Single-Node Repository (scripts)](https://github.com/coreos/coreos-kubernetes/tree/master/lib). All of this should happen on the local machine.

>NOTE: All the required scripts can be copied from the above mentioned repository (eg. wget https://raw.githubusercontent.com/coreos/coreos-kubernetes/master/lib/init-ssl and wget https://raw.githubusercontent.com/coreos/coreos-kubernetes/master/lib/init-ssl-ca), do note, that we do not take any responsibilites for those.

The IP.1 should be the Public IP of the CoreOS host (eg. 8.8.8.8), the IP.2 should be the Internal IP of the CoreOS host, which is defined in the Cloud-Config-file above.

``
mkdir ssl
./init-ssl-ca ssl
./init-ssl ssl apiserver controller IP.1=<PUBLIC_IP_HOST>,IP.2=10.3.0.1
./init-ssl ssl admin kube-admin
``

The generated files are then copied to the CoreOS host:

``
scp -r ssl core@<PUBLIC_IP_HOST>:/home/core
``

Then on the remote machine (CoreOS host), those files need to get moved to the correct location:

``
sudo mkdir -p /etc/kubernetes/ssl
sudo tar -C /etc/kubernetes/ssl -xf ssl/controller.tar
``

### Start Installation

To start the installation, the user_data from the [CoreOS-repository](https://raw.github.com/coreos/coreos-kubernetes/master/single-node/user-data) has to be adopted and copied to the remote host. Afterwards it can get executed and the install is basically done.

>NOTE: This file is adopted in some points, the adopted points are documented below.

``
# Port to open the API on to the external
export EXTERNAL_SSL_PORT=8443
``

Furthermore, to be able to provide SSL via kube-lego for our own services, we do need to change the ssl port in the section `kube-apiserver`. There the `hostPort` should be changed from 443 to 444. All of this is already done in the corresponding user-data in this repository.

The ADVERTISE_IP should be adopted to your personal needs as well.

>NOTE: This file is copied form the mentioned remote repository as well. The used version is for Kubernetes 1.5.4, and there could be changes in this file for future versions.

Ĉopy the user data to the remote host:

``
scp user_data core@<PUBLIC_IP_HOST>:/home/core
``

Move the User-Data and the ssl-key on the remote host to the correct location and execute the install script (user_data):

``
sudo mkdir -p /etc/kubernetes/ssl
sudo cp /home/core/ssl/ca.pem /etc/kubernetes/ssl
sudo cp /home/core/ssl/apiserver.pem /etc/kubernetes/ssl
sudo cp /home/core/ssl/apiserver-key.pem /etc/kubernetes/ssl

sudo mkdir -p /var/lib/coreos-kubernetes
sudo cp /home/core/user_data /var/lib/coreos-kubernetes/user_data
sudo chmod u+x /var/lib/coreos-kubernetes/user_data
sudo /var/lib/coreos-kubernetes/user_data
``

### Execute kubectl

To be able to execute kubectl on your local machine, you have to provide a valid kubeconfig. There is
one in this repository, but it needs to get adopted (PUBLIC_IP_HOST).

The kubectl client can be downloaded using the following command:

```
curl -O https://storage.googleapis.com/kubernetes-release/release/v1.6.1/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /usr/local/bin/kubectl
```

Afterwards, you are able to use the following commands to connect to your kubernetes cluster:

``
export KUBECONFIG="${KUBECONFIG}:$(pwd)/kubeconfig"
kubectl config use-context netcup
``

## Connect via Web-Browser

We did install the k8s Dashboard, and you can connect to it using your Web-Browser. Please use the
following URL:

```
https://YOUR_HOST:8443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/pod?namespace=default
```

### Pre-Requisites to use UI (SSL-Login-Certificate)

To be able to login to the UI, k8s expects you to have an certificate. This can be
generated using the already created ssl-certificates:

``
openssl pkcs12 -export -in ./ssl/admin.pem -inkey ./ssl/admin-key.pem -out ./ssl/admin.p12
``

Afterwards you have to import the generated file into chromium using the used password.

## Update CoreOS

``
sudo /usr/bin/systemctl unmask update-engine.service  
sudo /usr/bin/systemctl start update-engine.service  
sudo update_engine_client -update  
sudo /usr/bin/systemctl stop update-engine.service  
sudo /usr/bin/systemctl mask update-engine.service  
sudo reboot  
``

## Script Kiddies

sudo cp /usr/lib/systemd/system/sshd.socket /etc/systemd/system/sshd.socket
sudo vim /etc/systemd/system/sshd.socket (change listenstream port to eg 24)
sudo systemctl daemon-reload

see https://coreos.com/os/docs/latest/customizing-sshd.html
