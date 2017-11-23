# Kafka Cluster

## Introduction

Let's set up a Kafka Cluster here.

For the sake of the demo, we will use:

* Vagrant and Virtualbox to build VMs locally
* Ansible to provision them

> If you don't have these softwares, go get them.

```console
$ vagrant --version
Vagrant 2.0.0

$ VBoxManage --version
5.1.10r112026

$ ansible --version            
ansible 2.4.1.0
```

### The plan

* Step 1 will set up a single node where Kafka and Zookeeper will run together
* Step 2 will set up a cluster with 3 Zookeeper and 3 Kafka nodes
    * it is somehow the minimal set up as the distributed nature of the system implies a `quorum`, which means a majority of nodes must be up for the overall system to work. 
    Using 3 nodes allow us to have just one node potentially down at any point in time (say, for maintenance purposes)   

## Prerequisites

Just install the required tools as mentioned in introduction.

## Step1 - Standalone

Let's start with an instance of the couple Zookeeper / Kafka on a single node.


We will use here an Ubuntu Xenial box (16.04).
Let's declare it:
```console
$ vagrant init ubuntu/xenial64
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```
We can now run it:
```console
$ vagrant up
```
