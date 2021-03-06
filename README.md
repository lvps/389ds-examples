# 389DS examples

Get Vagrant, get Ansible, type these commands:

```shell
cd roles
git clone https://github.com/lvps/389ds-server.git
git clone https://github.com/lvps/389ds-replication.git
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

Two masters (consumer + supplier), replicating to each other with StartTLS.
SIMPLE authentication, TLS is enforced, both LDAPS and StartTLS are available.

First of all, generate certificates and keys:

```shell
cd ca
./cert.sh ldap1.example.local
./cert.sh ldap2.example.local
```

Then do:

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-multimaster
vagrant up
```

This will bring up two virtual machines:

| Vagrant name | Hostname            | IP         |
|--------------|---------------------|------------|
| mm1          | ldap1.example.local | 10.38.9.10 |
| mm2          | ldap2.example.local | 10.38.9.20 |

Use the Vagrant name for Vagrant commands, e.g. `vagrant ssh mm1` (useful to
check logs). Both machines can find each other via hostnames, but if you want
to connect to ldap1.example.local from the host machine, you'll need to add them
to your hosts file. There's an "hosts" file in this repo that gets deployed to
the VMs, you can copy the relevant lines from there.

`master-1.yml` and `master-2.yml` are the playbooks, you can tweak parameters there.

### Starting replication

*Replication is disabled by default*. Connect to ldap1.example.local, e.g. with
[Apache Directory Studio](https://directory.apache.org/studio/). Credentials are
`cn=Directory Manager` (bind DN) and `secret1` (password).

Navigate to `cn=agreement_with_ldap2_example_local,cn=replica,cn=dc\3Dexample\,dc\3Dlocal,cn=mapping tree,cn=config`,
e.g. with Apache Directory Studio right click on "Root DSE" on the left and
select "Go to DN..." (the `cn=config` tree is not visible by default).

You should see, among other attributes:

```
nsds5ReplicaEnabled: off
nsds5ReplicaLastUpdateStatus: Error (0) No replication sessions started since server startup
```

Change `nsds5ReplicaEnabled` to `on` and look at what `nsds5ReplicaLastUpdateStatus` says,
e.g. press F5 in Apache Directory Studio.

You will probably get an error in `nsds5ReplicaLastUpdateStatus`:

> Error (19) Replication error acquiring replica: Replica has different database generation ID, remote replica may need to be initialized (RUV error)

The solution is to add `nsds5BeginReplicaRefresh: start`, e.g. right click →
New Attribute → type "nsds5BeginReplicaRefresh" → Finish → type "start" and press enter.

This will instruct ldap1.example.com to take its database and push it
to ldap2.example.com.

The database at ldap2.example.com (i.e. everything under `dc=example,dc=local`)
will be *deleted without confirmation*: this is fine in a testing environment,
but in production you have to carefully decide which master will initialize
the others if you're adding more masters, that's why this part of the process
is manual.

The [Red Hat Directory Server Administration Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Directory_Server/10/)
(version 10) describes this procedure in more details in section 15.2.5.

Anyway, if all went well you should see `nsds5ReplicaLastUpdateStatus` say:

> Error (0) Replica acquired successfully: Incremental update started

This means that replication is taking place, it's not an error despite the word "Error".
However, we still need to enable replication in the reverse direction.

### Starting replication in the reverse direction

Connect to ldap2.example.local (`cn=Directory Manager`, `secret2`),
navigate to
`cn=agreement_with_ldap1_example_local,cn=replica,cn=dc\3Dexample\,dc\3Dlocal,cn=mapping tree,cn=config`
and set `nsds5ReplicaEnabled: on`. it should instantly say:

> Error (0) Replica acquired successfully: Incremental update started

and we're good to go!

To test that everything is working correctly, add and entry e.g. under
`ou=People,dc=example,dc=local` on one of the servers and watch it appear on the
other.

### A few notes

Since both servers were equally empty, initializing ldap2 before ldap1 would
have been the same. If ldap1 contained some entries under `dc=example,dc=local`,
initializing ldap1 would have replicated them to ldap2, while initializing from
ldap2 would have erased them.

Do *not* add `nsds5BeginReplicaRefresh: start` to the second master, it would
destroy and replace the database on the first master, it's not needed: when
ldap1 pushed its database to ldap2, they became identical so ldap2 can start
replicating without inconsistencies.

The "Install other certificate into NSS db" part in both playbooks installs
the certificate from the other server into NSS db of a server (e.g. ldap1
certificate into ldap2 NSS db): this is required because they are self-signed,
so servers wouldn't normally trust them and replication would fail.

The process of installing the "other" certificate if very fragile and doesn't take
rollovers very well, so if you replace on of the certificates for any reason you'll
probably have to delete it before provisioning with Ansible again.
Or you could just do `vagrant destroy && vagrant up`...

This certificate juggling is only needed because they are self-signed: if you use
a certificate from a trusted CA, it shouldn't be needed, but I haven't tested it
yet.

## Supplier and consumer

Two servers: one is a supplier, the other is a read-only consumer.

Generate certificates:

```shell
cd ca
./cert.sh ldap-ro-consumer.example.local
./cert.sh ldap-ro-supplier.example.local
```

Then start the VM:

```shell
export VAGRANT_VAGRANTFILE=Vagrantfile-roreplica
vagrant up
```

And you will get these:

| Vagrant name | Hostname                       | IP         |
|--------------|--------------------------------|------------|
| ro-supplier  | ldap-ro-supplier.example.local | 10.38.9.31 |
| ro-consumer  | ldap-ro-consumer.example.local | 10.38.9.32 |

Bind DN is `cn=Directory Manager`, password `supplier` or `consumer`. Use LDAPS (port 636) or StartTLS (port 389).

As usual, there's a manual step to start replication: connect to ro-supplier and
navigate to `cn=agreement_with_ldap-ro-consumer_example_local,cn=replica,cn=dc\3Dexample\,dc\3Dlocal,cn=mapping tree,cn=config`.

Replication is enabled by default (`nsds5ReplicaEnabled: on`), but you should
see the usual error in `nsds5ReplicaLastUpdateStatus`:

> Error (19) Replication error acquiring replica: Replica has different database generation ID, remote replica may need to be initialized (RUV error)

Add `nsds5BeginReplicaRefresh: start` and it should change to:

> Error (0) Replica acquired successfully: Incremental update succeeded

This operation destroyed the database ad ldap-ro-consumer.example.local and
replaced it with the database from ldap-ro-supplier.example.local, so replication
is now active.

A more detailed explanation of these steps and TLS configuration can be found in
the "Multi-master with 2 masters" example.

### Testing

To test if replcation is working, add an entry on ldap-ro-supplier.example.local,
e.g. to `ou=People,dn=example,dn=local`, and it will appear instantly on
ldap-ro-consumer.example.local.

However, if you try to add an entry on ldap-ro-consumer.example.local, it will
answer with a referral to the supplier server since consumer is read only.

For example, from the command line:

```shell
[vagrant@ldap-ro-consumer ~]$ ldapadd -ZZ -x -D "cn=Directory Manager" -W -f test.ldif
Enter LDAP Password:
adding new entry "cn=Foo,ou=People,dc=example,dc=local"
ldap_add: Referral (10)
        matched DN: dc=example,dc=local
        referrals:
                ldap://ldap-ro-supplier.example.local:389
```

and nothing gets actually added. You can access the command line on the VMs with
`vagrant ssh ro-consumer` like I did, or use your host machine by specifying
ldap-ro-consumer.example.local as the server, the rest of the command is the same.

`test.ldif` contained this:

```
dn: cn=Foo,ou=People,dc=example,dc=local
objectClass: person
cn: Foo
sn: Bar
```

Note that you have to disable certificate checking in the OpenLDAP tools to use
`ldapadd`, since it does not recognize the self signed certificate. This is fine
in testing, but don't do it in production!

Edit /etc/openldap/ldap.conf and add:

```
HOST ldap-ro-consumer.example.local
PORT 389
TLS_REQCERT NEVER
```

## Coming "soon"

I'll add these examples some day in the future. If you're interested, open an
issue and I'll try to add them sooner.

- 2 masters + 2 read-only replicas, the example from figure 15.2 in the Administration Guide
- Multi-master with 4 masters in a circle, the example from figure 15.3 in the Administration Guide
- Something with hubs, maybe? This will require modifications to the role...

## See also

If you're interested in running Keycloak or WSO2 IS on top of 389DS there's [another repository](https://github.com/WEEE-Open/sso) that can provide some hints. If you want some ACIs or custom schema files to deploy via Ansible, there's the [schema](https://github.com/weee-open/schema) repository.

## License

MIT unless otherwise noted (some of the involved software and Ansible roles have a different license)
