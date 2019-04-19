# 389DS examples

Get Vagrant, get Ansible, type these commands:

```shell
mkdir roles
cd roles
git clone https://github.com/lvps/389ds-server.git
git clone https://github.com/lvps/389ds-replication.git
cd ..
mkdir schema
git clone https://github.com/weee-open/schema.git
```

There are multiple Vagrantfiles, so you need to specify which one should be used, e.g.

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-single
vagrant up
```

or

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-multimaster
vagrant up
```
