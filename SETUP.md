# vTPM Client Certificate Injection PoC

## Setup a Host

- Install [Ubuntu Server 20.04.3](https://ubuntu.com/download/server) either on bare metal or under your favourite hypervisor. 

- We used the following:

```
Host: Ubuntu Server 20.04.3
Hypervisor: KVM using the virt-manager VMM
DevStack VM: Ubuntu Server 20.04.3
Disk: 30GB
```

In this guide we will refer to this host as the DevStack Host, whether or not it is running in a VM. 

## Install DevStack Wallaby or Xena

### Preparation

- Follow the steps from the [official guide](https://docs.openstack.org/devstack/wallaby/):

```
    sudo useradd -s /bin/bash -d /opt/stack -m stack

    echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

    sudo -u stack -i

    git clone https://opendev.org/openstack/devstack -b stable/wallaby

    cd devstack
    
    sudo apt remove python3-simplejson
    sudo apt remove python3-pyasn1-modules
    
```

- If stack.sh is being re-run, this may be needed:

```
    sudo apt remove -y openvswitch-common
    sudo rm /var/run/ovn/openvswitch
    sudo rm /var/run/ovn/ovn
```
- For Xena you may need to [change the OVN directory](https://stackoverflow.com/a/69010263)

```
Go to neutron_plugin folder, by default the folder is reside in the /opt/stack/devstack/lib directory.
open ovn_agent file with sudo privileges.
change line 116 which looks like this OVS_RUNDIR=$OVS_PREFIX/var/run/openvswitch you just have to change ovn by replacing of openvswitch. after change your line will become OVS_RUNDIR=$OVS_PREFIX/var/run/ovn now save the file.
```

### Installer Configuration

Ensure that there is no `localrc` file in your devstack source directory. 
This would cause the configuration parameters in `local.conf` to be skipped.

Create the configuration file:

```
    vi /opt/stack/devstack/local.conf   
```
```
[[local|localrc]]
 
enable_plugin barbican https://opendev.org/openstack/barbican stable/wallaby
enable_service rabbit mysql key
 
OS_USERNAME=admin
ADMIN_PASSWORD=asecurepass
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
LOGFILE=$DEST/logs/stack.sh.log
```

```
[[local|localrc]]
 
enable_plugin barbican https://opendev.org/openstack/barbican stable/xena
enable_service rabbit mysql key
 
OS_USERNAME=admin
ADMIN_PASSWORD=hg9sdoivanoscUHgui
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
LOGFILE=$DEST/logs/stack.sh.log
```

### Run Installer

```
./stack.sh
```

### Open Web Admin

Tunnel through to get remote external access to the web admin:

```
ssh -L 8001:localhost:80 <ip addr of Devstack Host>
```


## Enable swtpm for vTPM Support

Refer to [the official Emulated Trusted Platform Module (vTPM) guide](https://docs.openstack.org/nova/victoria/admin/emulated-tpm.html) and perform the following steps.

Barbican should be available as it will have been deployed as part of the installation process above. Barbican is the key manager service and it is used to encrypt the vTPM files at rest on the host.

### Install tpm2_tools

```
apt install tpm2-tools
```

### Install libtpms

As the stack user in /opt/stack:

``` 
git clone https://github.com/stefanberger/libtpms.git -b stable-0.9
cd libtpms
./autogen.sh
./configure --prefix=/opt/sdk/libtpms/09
make
sudo make install
sudo cp ./libtpms.pc /usr/share/pkgconfig/
```

### Install swtpm 


```
sudo apt install libtasn1-6-dev
sudo apt install libjson-glib-dev
sudo apt install expect
sudo apt install libseccomp-dev
sudo apt install libgnutls28-dev
sudo apt install gnutls-bin

git clone https://github.com/stefanberger/swtpm.git -b stable-0.6
cd swtpm
./autogen.sh
./configure --prefix=/opt/apps/swtpm/06 --with-gnutls
make
sudo make install
sudo ln -s /opt/apps/swtpm/06 /opt/apps/swtpm/current
sudo echo "/opt/apps/swtpm/current/lib/swtpm" > /etc/ld.so.conf.d/custompaths.conf
sudo cp /opt/apps/swtpm/current/bin/swtpm* /usr/bin/

sudo ldconfig

```

Modify the AppArmor configuration for libvirt-qemu.

`vi /etc/apparmor.d/abstractions/libvirt-qemu`

Find the following section:

``` 
  # swtpm
  /{usr/,}bin/swtpm rmix,
  /usr/{lib,lib64}/libswtpm_libtpms.so mr,
  /usr/lib/@{multiarch}/libswtpm_libtpms.so mr,
```

Add the following lines:
```
  /opt/apps/swtpm/06/lib/swtpm/libswtpm_libtpms.so.0.0.0 mr,
  /opt/sdk/libtpms/09/lib/libtpms.so.0.9.1 mr,
```

Reload the AppArmor service:

`sudo systemctl reload apparmor.service`

#### Modify Nova conf 

Add `swtpm_enabled` to `/etc/nova/nova-cpu.conf`

```
[libvirt]
live_migration_uri = qemu+ssh://stack@%s/system
cpu_mode = none
virt_type = kvm
swtpm_enabled = True
```

###Restart Nova Compute and Verify Status

``` 
systemctl restart devstack@n-cpu.service
systemctl status devstack@n-cpu.service
```

```
. ./openrc admin
```

## Create a vTPM Enabled Instance

### Add vTPM Metadata to the cirros Image

[Add the following parameters to an image:](https://docs.openstack.org/nova/wallaby/admin/emulated-tpm.html#configuring-a-flavor-or-image)

Select Project->Compute->Images, select the cirros image and select Update Metadata from the Launch drop down on the right

![DevStack Image Metadata](https://mode51.software/downloads/doc/images/devstackimages.png)

Add the following image metadata by entering the name in the Custom input text box, click +, then on the right enter the values:
	
```
Name: hw_tpm_version
Value: 2.0	

Name: hw_tpm_model
Value: tpm-crb
```

Remove the following empty elements:

```
owner_specified.openstack.md5
owner_specified.openstack.sha256
```

Then Save

![DevStack Image Metadata](https://mode51.software/downloads/doc/images/devstacklaunchcirrosvtpmmeta1.png)

### Create a New Security Group

Navigate to `Project->Network->Security Groups`

Select `Create Security Group`

In the popup enter a `Name` and select `Create Security Group`

Under the `Manage Security Group` page click `Add Rule`, select `All TCP`, confirm that the `Direction` is `Ingress` and then `Add`

### Create a Cirros Instance with a vTPM Attached

In the webadmin:

1. Navigate to `Project->Compute->Instances`

2. Select `Launch Instance`

3. Enter an instance name under `Details`

![DevStack Image Metadata](https://mode51.software/downloads/doc/images/devstacklaunchcirrosvtpm1.png)

4. Under the next page, `Source`, select the cirros image so that it is listed under `Allocated`

![DevStack Image Metadata](https://mode51.software/downloads/doc/images/devstacklaunchcirrosvtpm2.png)

5. Under `Flavour` select `m1.nano` so that it is listed under `Allocated`

6. Under `Networks` add the instance to the shared net

7. In `Security Groups` select the default group under `Allocated` so that it is released into `Available`. Then, select the new group you created earlier from `Available` so that it is listed as `Allocated`

8. Select `Launch Instance`

9. SSH into the Instance or Click on your instance and select the `Console` tab. A tunnel may be needed to access port 6080 on the internal IP, and in this case you may need to copy the link to the console and change the IP to 127.0.0.1:

```
ssh -L 6080:<instance IP addr>:6080 <Devstack Host IP addr>
```
9. Login using the reported username and password on the console

10. Verify the presence of the vTPM:

```
ls -al /dev/tpm0
```

11. Shutdown the Cirros Instance


### Configure libvirt to Support a Persistent vTPM

By default the vTPM in libvirt 6 (used by Wallaby) is deleted on shutdown and recreated on startup. 
In libvirt 7 (used by Xena) the vTPM can be persisted with configuration.

#### Persistent vTPM for DevStack Wallaby

libvirt 6.0.0 needs a small patch to enable persistent vTPMs. If you are using Wallaby, perform the following steps:

```
sudo systemctl stop libvirtd
sudo cp /usr/lib/x86_64-linux-gnu/libvirt/connection-driver/libvirt_driver_qemu.so ./libvirt_driver_qemu.org
sudo cp libvirt_driver_qemu.so /usr/lib/x86_64-linux-gnu/libvirt/connection-driver/libvirt_driver_qemu.so
sudo systemctl start libvirtd
```

#### Persistent vTPM for DevStack Xena

libvirt 7.0.0 supports a new XML configuration parameter that makes the vTPM persistent.

TODO

### Import an Ubuntu Server Image

Download the Ubuntu cloud image 20.04:

```
https://cloud-images.ubuntu.com/focal/20211015/
```

Open `Compute->Images`

Select `Create Image`

Click `File` and select the Ubuntu cloud image downloaded previously
Select `QCOW2` as the image format
Select `Next`

In the meta data, add the TPM parameters in the same way as for the Cirros image:

```
Name: hw_tpm_version
Value: 2.0	

Name: hw_tpm_model
Value: tpm-crb
```

Select `Create Image` and the Ubuntu Cloud server image should appear in the list of images.


### Create an Ubuntu Server Cloud Instance

Install DNS on the Devstack Host:

```
sudo apt-get install bind9
```

Navigate to `Project->Instances`

Select `Launch Instance`

Name the New Instance

Click `Next`

Under `Available` select the Ubuntu Cloud Server image so that it is listed as `Allocated`

Click `Next`

Under `Available` select a flavour such as `m1.small` so that it is listed as `Allocated`

Click `Next`

In Networks under `Available` select `shared` so that it is listed as `Allocated` 

Click `Next` twice

In `Security Groups` select the default group under `Allocated` so that it is released into `Available`. Then, select the new group you created earlier from `Available` so that it is listed as `Allocated`

Click `Next`

Select `Create Key Pair`

Enter a `Key Pair Name` and select `SSH Key` as the key type

Select `Creae Keypair` and copy the private key to the clipboard

Save the private key on the Devstack host as it will be used to SSH into the new instance:

```
su - testuser
mkdir .ssh
chmod 500 .ssh
vi .ssh/`id_ubucloudkey`
paste the key material and save
chmod 400 .ssh/id_ubucloudkey
```

Select `Done`

Select `Launch Instance`

#### Connect to the Ubuntu VM

Navigate to Projects -> Network -> Routers

Select `Create Router`

Enter a name and select the public network under `External Network`

`Enable Admin State` and `Enable SNAT` should already be selected.

Navigate to Projects -> Network -> Floating IPs

Select `Allocate IP to Project`

Select `public` under Pool and `Allocate IP`

Select `Associate IP`

Under `port` select the IP address of the Ubuntu VM. 

The VM should now be accessible from the Devstack host.

Use the generated key to access via ssh:

```
ssh -i ./.ssh/id_ubucloudkey ubuntu@<ip addr>
```

Confirm the vTPM is enabled:

```
ls -al /dev/tpm0
```

Enable DNS:

``` 
vi /etc/resolv.conf
```
Change the nameserver to be the IP of the Devstack Host and save.

Shutdown the VM.

## Manage the vTPM on the Devstack Host

### Extract the Instance's vTPM on the Hypervisor 

Find the instance id:

```
su - stack
cd devstack
export OS_PROJECT_NAME="admin"
. ./openrc admin
nova list
```
+--------------------------------------+----------------------+---------+------------+-------------+------------------------------------+
| ID                                   | Name                 | Status  | Task State | Power State | Networks                           |
+--------------------------------------+----------------------+---------+------------+-------------+------------------------------------+
| 1b277916-c183-41ec-8676-0c68f76b0ec4 | alpinevirt           | SHUTOFF | -          | Shutdown    | public=172.24.4.232, 2001:db8::133 |
| ca6c0d47-3061-4806-8998-441b638b88e9 | ubuntucloudserv20.04 | SHUTOFF | -          | Shutdown    | shared=192.168.233.97, 172.24.4.73 |
+--------------------------------------+----------------------+---------+------------+-------------+------------------------------------+

In this example the instance ID  we will use is for ubuntucloudserv20.04:

```
ca6c0d47-3061-4806-8998-441b638b88e9
```

#### Lookup the vTPM's at rest encryption key

List the current secrets:

```
openstack secret list
```

Find the secret by looking for the instance ID in the name field:

```
+------------------------------------------------------------------------------------+---------------------------------------------------------------+---------------------------+--------+-----------------------------------------+-----------+------------+-------------+------+------------+
| Secret href                                                                        | Name                                                          | Created                   | Status | Content types                           | Algorithm | Bit length | Secret type | Mode | Expiration |
+------------------------------------------------------------------------------------+---------------------------------------------------------------+---------------------------+--------+-----------------------------------------+-----------+------------+-------------+------+------------+
| http://192.168.122.118/key-manager/v1/secrets/556bd168-9386-40db-95e5-8c69f29a61c4 | vTPM secret for instance 1b277916-c183-41ec-8676-0c68f76b0ec4 | 2021-10-24T00:30:44+00:00 | ACTIVE | {'default': 'application/octet-stream'} | None      | None       | passphrase  | None | None       |
| http://192.168.122.118/key-manager/v1/secrets/d6a852e9-9fe4-42af-977b-0785866c7af5 | vTPM secret for instance ca6c0d47-3061-4806-8998-441b638b88e9 | 2021-10-24T00:50:01+00:00 | ACTIVE | {'default': 'application/octet-stream'} | None      | None       | passphrase  | None | None       |
+------------------------------------------------------------------------------------+---------------------------------------------------------------+---------------------------+--------+-----------------------------------------+-----------+------------+-------------+------+------------+
```

In this example the secret ID is identified in the first record. Check the `Name` field to find your instance ID.

Copy the Secret href for your and run the following command:

```
openstack secret get http://192.168.122.118/key-manager/v1/secrets/d6a852e9-9fe4-42af-977b-0785866c7af5 --payload
```

Copy this payload value into a file on the hypervisor.

```
sudo - stack
mkdir ~/keys
vi ~/keys/instancedec.key
truncate -s 512 ~/keys/instancedec.key
```

#### Mount the vTPM proxy on the Hypervisor

The vTPM proxy provides access to the vTPM on the hypervisor using standard tpm2_tools commands.

The Instance with the vTPM attached MUST be shutdown/suspended before the vTPM can be mounted by the hypervisor.

```
sudo modprobe tpm_vtpm_proxy
sudo swtpm chardev --tpm2 --vtpm-proxy --tpmstate dir=/var/lib/libvirt/swtpm/ca6c0d47-3061-4806-8998-441b638b88e9/tpm2 --key pwdfile=/opt/stack/keys/instancedec.key,mode=aes-256-cbc,format=binary &
```

```
sudo apt install tpm2-abrmd
```

#### Extract the EK

This tpm2 command will list the contents of the vTPM's NV RAM:
```
sudo tpm2_nvreadpublic
```
Sample output:

```
0x1c00002:
name: 000bf92260ecedb24149d8ac894c004577e65ca46dd7b50f03ce158121cfbec6f58a
hash algorithm:
friendly: sha256
value: 0xB
attributes:
friendly: ppwrite|writelocked|writedefine|ppread|ownerread|authread|no_da|written|platformcreate
value: 0x1280762
size: 1062
```

0x1c00016 and 0x1c08000 will also be visible.

[TPM2 NV Index 0x1c00002 is the TCG specified location for RSA-EK-certificate](https://tpm2-software.github.io/2020/06/12/Remote-Attestation-With-tpm2-tools.html#simple-attestation-with-tpm2-tools).

Look for the ID (0x1c00002) and the size (1062) and read the certificate:

```
sudo tpm2_nvread --hierarchy owner --size 1062 --output outcert.pem 0x1c00002
```

Use openssl to check the certificate's structure:

```
openssl x509 -in ./outcert.pem -inform DER -text
```

### Build OpenSSL 3.0 with CMPv2 Support on the Devstack Host

```
git clone https://github.com/openssl/openssl.git -b openssl-3.0
cd openssl
./config --prefix=/opt/sdk/openssl/300 enable-trace enable-ssl-trace enable-ktls
sudo make install
sudo cp ./*.pc /usr/share/pkgconfig
sudo ln -s /opt/sdk/openssl/300 /opt/sdk/openssl/current
```

### Add openssl v3 to PATH:

```
export LD_LIBRARY_PATH=/opt/sdk/openssl/current/lib64
export PATH=/opt/sdk/openssl/current/bin:$PATH
```

### Build OpenSSL 3.0 TPM Provider

```
su - stack
sudo apt install libtss2-dev
git clone https://github.com/tpm2-software/tpm2-openssl.git -b 1.0.0
cd tpm2-openssl
sudo apt install autoconf-archive
sudo apt install pkg-config
sudo apt install libtool
./bootstrap
PKG_CONFIG_PATH=/usr/share/pkgconfig LD_LIBRARY_PATH=/opt/sdk/openssl/current/lib64 ./configure --disable-op-digest
make
sudo make install

```

### Create TPM EK and AK Keys

Ignore the warnings when the following commands are run:

```
#sudo tpm2_createak -C 0x81010001 -c ak.ctx -G rsa -g sha256 -s rsassa

su
tpm2_createek -c ek.ctx -G rsa -u ek.pub
tpm2_nvreadpublic
tpm2_createak -C ek.ctx -c ak.ctx -G rsa -g sha256 -s rsassa
tpm2_nvreadpublic
tpm2_evictcontrol -C o -c ak.ctx 0x81010002
tpm2_nvreadpublic
tpm2_readpublic -c ak.ctx -f pem -o ak.pem > ak.yaml
```

### Use the TPM2 ABRMD Resource Manager

```
sudo apt install libtss2-tcti-tabrmd0
rm /usr/lib/x86_64-linux-gnu/libtss2-tcti-default.so
ln -s /usr/lib/x86_64-linux-gnu/libtss2-tcti-tabrmd.so.0 /usr/lib/x86_64-linux-gnu/libtss2-tcti-default.so
```

### Acquire a Signed Certificate for the TPM AK from a CA Using CMPv2

A passphrase is used to issue an initial certificate that can subsequently be used for authentication.

Please use your passphrase. 
```
openssl cmp -config /opt/sdk/openssl/current/ssl/openssl.cnf -provider tpm2 -provider default -propquery ?provider=tpm2 -cmd ir -server https://demo.one.digicert.com/iot/api/v1/cmp/IOT_01b40786-a875-4cd0-ba8e-1dce0af2e830 -ref dd0c5b91-127c-4d86-91a8-4dfa8583c58e -secret pass:1234 -recipient "/CN=mode51.software" -key handle:0x81010002 -subject "/CN=DevStackTest" -cacertsout ./capubs.pem -certout ./cl_cert.pem -tls_used -verbosity 8
```

### Permit Any Cert Signed with the BT Issuing CA to Authenticate to DigiCert

![DevStack Image Metadata](https://mode51.software/downloads/doc/images/digicertenrollbtissuingcaconf.png)


### Use the Signed Certificate to Issue New Certificates

```
/opt/sdk/openssl/current/bin/openssl cmp -config /opt/sdk/openssl/current/ssl/openssl.cnf -provider tpm2 -provider default -propquery ?provider=tpm2 -cmd cr -server https://demo.one.digicert.com/iot/api/v1/cmp/IOT_01b40786-a875-4cd0-ba8e-1dce0af2e830 -ref dd0c5b91-127c-4d86-91a8-4dfa8583c58e -key handle:0x81010002 -certout ./cl_cert2.pem -cert ./cl_cert.pem -tls_used -verbosity 8 -trusted ./capubs.pem -unprotected_errors
```

### Store Client Cert in vTPM NV RAM

```
ls -al cl_cert.pem

rw-r--r-- 1 root root 1858 Oct  5 23:33 cl_cert.pem
```

Reserve space where the size `-s` is observed in ls -al
```
tpm2_nvdefine 0x1500016 -C o -s 1858 -a "ownerread|ownerwrite|policywrite"
```

Write the signed client cert into the vTPM
```
tpm2_nvwrite 0x1500016 -C o -i cl_cert.pem 
```

Confirm that the signed client cert can be read from the vTPM:
```
tpm2_nvread 0x1500016 -C o
```

Also store the public root and issuing CA certificates. The vTPM records are limited to 2048 bytes and so the certificates need to be split into two files and loaded into two separate slots.

Split the certificates in capubs.pem:
```
cat capubs.pem |awk 'split_after==1{n++;split_after=0} /-----END CERTIFICATE-----/ {split_after=1} {print > "cacert" n ".pem"}'
```

`cert.pem` and `cert1.pem` will be available for a file with two certificates.

Again, find the size for each certificate for the -s switch.

The limit is 2048 bytes, so the certificate may need to be gzipped before being written into the vTPM.

```
gzip cacert.pem
ls -al cacert.pem.gz
tpm2_nvdefine 0x1500017 -C o -s 1416 -a "ownerread|ownerwrite|policywrite"
tpm2_nvwrite 0x1500017 -C o -i cacert.pem.gz
tpm2_nvread 0x1500017 -C o

tpm2_nvdefine 0x1500018 -C o -s 1667 -a "ownerread|ownerwrite|policywrite"
tpm2_nvwrite 0x1500018 -C o -i cacert1.pem 
tpm2_nvread 0x1500018 -C o

```

### Unmount the vTPM

```
ps waux | grep swtpm
kill -HUP <pid>
systemctl stop tpm2-abrmd
```



### Start the Ubuntu VM

In the webadmin start the Ubuntu VM.

ssh in from the Devstack Host:

``` 
ssh -i ./.ssh/id_ubucloudkey ubuntu@<ip addr>
sudo apt update
```

### Inspect the vTPM Certificate Inside the VM

Install the TPM2 tools inside the VM, as root:
```
apt install tpm2-tools
apt install libtss2-tcti-tabrmd0
apt install libtss2-dev
rm /usr/lib/x86_64-linux-gnu/libtss2-tcti-default.so
ln -s /usr/lib/x86_64-linux-gnu/libtss2-tcti-tabrmd.so.0 /usr/lib/x86_64-linux-gnu/libtss2-tcti-default.so
apt install tpm2-abrmd
```

### Install openssl v3 with CMPv2

```
apt install gcc
apt install make
git clone https://github.com/openssl/openssl.git -b openssl-3.0
cd openssl
./config --prefix=/opt/sdk/openssl/300 enable-trace enable-ssl-trace enable-ktls
make install
cp ./*.pc /usr/share/pkgconfig
ln -s /opt/sdk/openssl/300 /opt/sdk/openssl/current

```

### Add openssl v3 to PATH:

```
export LD_LIBRARY_PATH=/opt/sdk/openssl/current/lib64
export PATH=/opt/sdk/openssl/current/bin:$PATH
```

### Build OpenSSL 3.0 TPM Provider

```
cd /home/ubuntu
git clone https://github.com/tpm2-software/tpm2-openssl.git -b 1.0.0
cd tpm2-openssl
sudo apt install autoconf-archive
sudo apt install pkg-config
sudo apt install libtool
./bootstrap
PKG_CONFIG_PATH=/usr/share/pkgconfig LD_LIBRARY_PATH=/opt/sdk/openssl/current/lib64 ./configure --disable-op-digest
make install

```

## Virtual Machine Access to vTPM

### Extract the Client Cert

```
tpm2_nvread 0x1500016 -C o -o cl_cert.pem
cat cl_cert.pem
```

##Extract the CA Certs

``` 
tpm2_nvread 0x1500017 -C o -o cacert.pem.gz
gunzip cacert.pem.gz
tpm2_nvread 0x1500018 -C o -o cacert1.pem
cat cacert.pem cacert1.pem > capubs.pem
```

###Add openssl v3 to PATH:

```
export LD_LIBRARY_PATH=/opt/sdk/openssl/current/lib64
export PATH=/opt/sdk/openssl/current/bin:$PATH
```

### Issue a new Certificate

After all of these steps it should now be possible to request new certificates from the CA 
using the client certificate and private key injected into the vTPM by the hypervisor:

``` 
/opt/sdk/openssl/current/bin/openssl cmp -config /opt/sdk/openssl/current/ssl/openssl.cnf -provider tpm2 -provider default -propquery ?provider=tpm2 -cmd cr -server https://demo.one.digicert.com/iot/api/v1/cmp/IOT_01b40786-a875-4cd0-ba8e-1dce0af2e830 -ref dd0c5b91-127c-4d86-91a8-4dfa8583c58e -key handle:0x81010002 -certout ./cl_cert2.pem -cert ./cl_cert.pem -tls_used -verbosity 8 -trusted ./capubs.pem -unprotected_errors
```




## Troubleshooting

If the tpm0 device isn't at /dev/tpm0, HUP swtpm, stop the tpm2-abrmd service, then start swtpm and start the tpm2-abrmd service.
