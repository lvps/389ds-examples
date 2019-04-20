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

There are multiple Vagrantfiles, so you need to specify which one should be used.

## Single

A single 389DS istance, on CentOS, in a Vagrant VM.

Complete with self-signed TLS certificate, which you'll have to generate:

```shell
cd ca
./cert.sh ldaptest.example.local
```

This will create `ca/ldaptest_example_local_cert.pem` and `ldaptest_example_local.key`.

Then start the VM:

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-single
vagrant up
```

it will be provisioned automatically with Ansible and you'll get a running instance
at 10.38.9.99 (ldaptest.example.local, you can add it to your hosts file).

Port 389 and 636 are exposed, StartTLS is enabled, TLS is enforced
(you can't bind without StartTLS or TLS), username is `cn=Directory Manager` and
password is `secret`. Check out `single.yml` for the other parameters. You can
tweak it and run `vagrant provision` to apply changes.

## Multi-master with 2 masters

Work in progress, not ready yet.

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-multimaster
vagrant up
```
