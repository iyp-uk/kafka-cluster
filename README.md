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

### Installing Java

Java is required for Kafka and Zookeeper to run.

But it doesn't come by default on our box:
```console
$ vagrant ssh -c "which java"
Connection to 127.0.0.1 closed.
```
So we will install it.
For it, let's use Ansible :)

Let's clean up our auto-generated `Vagrantfile` as the only useful information is `config.vm.box = "ubuntu/xenial64"` so what we have:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/xenial64"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbook.yml"
    ansible.extra_vars = {
        ansible_python_interpreter: "/usr/bin/python3",
    }
  end
end
```
And add this `playbook.yml` which will be our entry point for Ansible provisioning.

```console
$ mkdir ansible
$ touch ansible/playbook.yml
```
We will use the [Java role from Geerlingguy](https://github.com/geerlingguy/ansible-role-java)
```console
$ ansible-galaxy install  -p ./ansible/roles geerlingguy.java
- downloading role 'java', owned by geerlingguy
...
- geerlingguy.java (1.7.4) was installed successfully
```
We can see that it will install `openjdk-8-jdk` as specified in `ansible/roles/geerlingguy.java/vars/Ubuntu-16.04.yml`.
```console
$ cat ansible/roles/geerlingguy.java/vars/Ubuntu-16.04.yml
---
# JDK version options include:
#   - java
#   - openjdk-8-jdk
#   - openjdk-9-jdk
__java_packages:
  - openjdk-8-jdk
```

Let's add it to our `ansible/playbook.yml`):
```yml
- hosts: all
  become: true
  roles:
    - role: geerlingguy.java
```

So let's provision the machine to apply changes:
```console
$ vagrant provision
```

Check if it's installed and in which version:
```console
$ vagrant ssh -c "which java"  
/usr/bin/java
```
It's there, and:
```console
$ vagrant ssh -c "java -version"
openjdk version "1.8.0_151"
OpenJDK Runtime Environment (build 1.8.0_151-8u151-b12-0ubuntu0.16.04.2-b12)
OpenJDK 64-Bit Server VM (build 25.151-b12, mixed mode)
```
in the version we expected. OK.

### Installing Zookeeper

Same as for Java, let's not reinvent the wheel and use an existing role.

```console
$ ansible-galaxy install  -p ./ansible/roles AnsibleShipyard.ansible-zookeeper
```
It comes with a [bunch of defaults](ansible/roles/AnsibleShipyard.ansible-zookeeper/defaults/main.yml) which we will customise slightly for now (the default mirror returned `404` and the version isn't the latest one):
```yaml
- name: Installing ZooKeeper
  hosts: all
  sudo: yes
  roles:
    - role: AnsibleShipyard.ansible-zookeeper
      zookeeper_version: 3.4.10
      zookeeper_url: http://www.mirrorservice.org/sites/ftp.apache.org/zookeeper/zookeeper-{{zookeeper_version}}/zookeeper-{{zookeeper_version}}.tar.gz
```
See if this runs:
```console
$ vagrant ssh -c "ps -ef | grep java"
zookeep+  8316     1  0 13:50 ?        00:00:00 java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /opt/zookeeper-3.4.10/bin/../build/classes:/opt/zookeeper-3.4.10/bin/../build/lib/*.jar:/opt/zookeeper-3.4.10/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/zookeeper-3.4.10/bin/../lib/slf4j-api-1.6.1.jar:/opt/zookeeper-3.4.10/bin/../lib/netty-3.10.5.Final.jar:/opt/zookeeper-3.4.10/bin/../lib/log4j-1.2.16.jar:/opt/zookeeper-3.4.10/bin/../lib/jline-0.9.94.jar:/opt/zookeeper-3.4.10/bin/../zookeeper-3.4.10.jar:/opt/zookeeper-3.4.10/bin/../src/java/lib/*.jar:/opt/zookeeper-3.4.10/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /opt/zookeeper-3.4.10/bin/../conf/zoo.cfg
```
Yay! It runs.

### Installing Kafka

```console
$ ansible-galaxy install  -p ./ansible/roles jaytaylor.kafka
```

and in `ansible/playbook.yml`:

```yaml
- name: Installing Kakfa
  hosts: all
  become: yes
  roles:
    - role: onaio.kafka
      kafka_version: 1.0.0
      kafka_hosts: "{{ ansible_fqdn }}:9092"
      zookeeper_hosts: "{{ ansible_fqdn }}:2181"
      kafka_heap_opts: "-Xmx521M -Xms512M"
```

> We've voluntary reduced heap opts here just so that it first in the VM. It's just a demo after all, shall be enough.

Let's give it a try:

```console
$ vagrant ssh
$ KAFKA_ROOT=/opt/kakfa/

## Check the topics already there (we expect none of course!)
$ $KAFKA_ROOT/bin/kafka-topics.sh --list --zookeeper localhost:2181
$ 

## So let's add one:
$ $KAFKA_ROOT/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test-one
Created topic "test-one".

## Add some messages:
$ $KAFKA_ROOT/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-one
>Hello world!
>Sushi
> ^D

## And retrieve them from the console consumer:
$ $KAFKA_ROOT/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
Hello world!
Sushi
^C
```

> Had to make some changes to the `onaio.kafka` role so that it can cope with the version `1.0.0` version of Kafka.
