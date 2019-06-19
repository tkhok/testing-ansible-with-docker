# Ansible Test with docker containers

There is a need to use ansible with docker container sometimes because of environment it's hard to install ansible directly.  
This usages and settings for docker container will help you as one of the solution.
This work involves almost empty containers only installed ssh which can be target servers for testing by ansible container.

### Description

There are two containers in default:

* Ansible on CentOS7
* A server installed open-ssh and launched SSH on CentOS7 connected with Ansible container

#### Caveat

All settings and usages must be limited to development environments. SSH should not be used root login and password in any case.

#### Ansible

Requirements:

* CentOS7
* SSH
* python 2.7

DockerFile-testing-ansible

```sh
# OS
FROM centos:centos7

# Base
RUN yum install -y gcc libffi-devel python-devel openssl-devel epel-release sshpass
RUN yum install -y openssh-server openssh-clients
RUN curl -kL https://bootstrap.pypa.io/get-pip.py | python

# Ansible Install
RUN pip install ansible
```

docker build & run

```sh
docker build -f ./Dockerfile-testing-ansible -t testing-ansible .
docker run -itd testing-ansible
```



#### SSH

Requirements:

* CentOS7
* SSH

DockerFile-simple-ssh

```sh
# OS
FROM centos:centos7

# Install packages
RUN yum -y update && yum clean all
RUN yum install -y which
RUN yum install -y wget
RUN yum install -y tar
RUN yum install -y vim
RUN yum install -y git
RUN yum -y install openssh-server openssh-clients

ENV LOGIN_PATH password
RUN echo $LOGIN_PATH | passwd --stdin root

# Modifying ssh config
RUN ssh-keygen -t rsa -N "" -f /etc/ssh/ssh_host_rsa_key
RUN sed -ri 's/^UsePAM yes/UsePAM no/' /etc/ssh/sshd_config
RUN sed -ri 's/^#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config

# launch sshd
EXPOSE 10022
CMD ["/usr/sbin/sshd", "-D"]
```

docker run & build

```sh
docker build -f ./Dockerfile-simple-ssh -t simple-ssh .
docker run -itd simple-ssh
```



#### Docker compose

docker-compose.yml

```yaml
# docker-compose for testing-ansible
version: "2"
services:
    ssh:
        build:
            context: .
            dockerfile: ./Dockerfile-simple-ssh
        ports:
            - "10022:22"
        tty: true

    ansible:
        build:
            context: .
            dockerfile: ./Dockerfile-testing-ansible
        volumes:
            - ./ansible:/ansible
        working_dir: /ansible
        depends_on:
            - ssh
        tty: true
```

Launching containers

```sh
docker-compose up -d
```

### Usage

#### ssh

```sh
docker exec -it {container-of-ansible} bash
ssh -l root@ssh
# insert password
```

#### Testing ping playbook

##### inventory file

```sh
# ansible/inventory
[base]
ssh
[all:vars]
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass=password
```

##### playbook

```yaml
# ansible/ping.yml
- hosts: base
  tasks:
    - name: ping
      ping:
```

##### Testing

```sh
# make know_hosts
docker exec -it {ansible-container} ssh -l root ssh
# execute playbook
docker exec -it {ansible-container} ansible-playbook ping.yml -i inventory
```

