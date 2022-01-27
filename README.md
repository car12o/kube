# kube bare metal
Bare metal [kubernetes](https://kubernetes.io/) cluster using [Vagrant](https://www.vagrantup.com/).

## Requirements
- [Vagrant](https://www.vagrantup.com/)
- [VirtualBox](https://www.virtualbox.org/)

## Quick start
### Configuration
1. Create config file
```bash
cp env.example.yml env.yml
```
2. Update network variable `MASTER_IP` & `NETWORK_CIDR`
```bash
// MASTER_IP & NETWORK_CIDR should be in virtualbox network range.
// Find virtualbox network with the following command on `vboxnet` net address.
ip -a
```
### Bootstrap cluster
```bash
vagrant up
```
### Setup kubectl
```bash
// get config from master node
vagrant ssh -c "cat \$HOME/.kube/config"
// append configuration to kube config on the host
nano $HOME/.kube/config
```
### Install nginx ingress
```bash
// after cluster bootstrap
vagrant up --provision-with=ingress
```
### Pause cluster
```bash
vagrant halt
```
### Tear down cluster
```bash
vagrant destroy -f
```
