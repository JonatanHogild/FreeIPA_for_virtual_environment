# FreeIPA for a virtual environment

<img width="" src="https://github.com/JonatanHogild/FreeIPA_for_virtual_environment/blob/main/images/freeipa.png"> 


**FreeIPA for a virtual enviroment**
<br>**Authors:** _<a href="https://github.com/JonatanHogild">Jonatan Högild</a> and <a href="https://github.com/Filipanderssondev">Filip Andersson</a>_ <br>
24-02-2026<br>

## Abstract
Implementation of the identity and authentication management solution FreeIPA for a virtual production-environment.

## Table of Contents

1. [Introduction](#introduction)
2. [Goals and Objectives](#goals-and-objectives)
3. [Method](#method) <br>
4. [Target Audience](#target-audience)
5. [Document Status](#document-status)
6. [Disclaimer](#disclaimer)
7. [Scope and Limitations](#scope-and-limitations)
8. [Environment](#environment)
9. [Acknowledgments](#acknowledgments)
10. [Implementation](#implementation) <br>
    10.1 [Prepare a new VM](#31-prepare-a-new-vm) <br>
    10.2 [DNS](#32-dns) <br>
    10.3 [FreeIPA installation](#33-freeipa-installation) <br>
    10.4 [Chrony/NTP](#34-chronyntp) <br>
    10.5 [Firewall configuration](#35-firewall-configuration) <br>
    10.6 [Set up FreeIPA-clients with Ansible](#36-set-up-freeipa-clients-with-ansible) <br>
    10.7 [Verify that FreeIPA works so far](#37-verify-that-freeipa-works-so-far) <br>
    10.8 [IPA users](#38-ipa-users) <br>
    10.9 [IPA groups](#39-ipa-groups) <br>
    10.10 [HBAC](#310-hbac) <br>
    10.11 [sudo rules](#311-sudo-rules) <br>
    10.12 [SSH key management](#312-ssh-key-management) <br>
    10.13 [Break-glass admin](#313-break-glass-admin) <br>
    10.14 [Subordinate UID/GID](#314-subordinate-uidgid) <br>
11. [Conclusion](#conclusion)
12. [References](#references) 

## Introduction
**Welcome!**
In this lab, we'll dive into [FreeIPA](https://www.freeipa.org/), an IAM-solution for Linux. It is no small task to manage hundreds, or even thousands of users across an enterprise. Identity and Access Management (IAM) is a framework of policies and technologies brought together to handle this issue. With this implementation of FreeIPA, we set out to manage identities and authentication in our small virtual environment as if it were a large enterprise environment. A lot will be covered in this lab: DNS, installation of FreeIPA-servers and clients, IPA users and groups, host-based access control, sudo rules, and much more. This is the fifth project <a href="https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc/blob/main/Extra/Mermaid/Projects.md">in a series of projects</a>, with the goal of setting up a complete virtualized, automated, and monitored IT-Enviroment as a part of our internship at [The Swedish Meteorological and Hydrological Institute (SMHI)](https://www.smhi.se/en/about-smhi). <br>

## Goals and Objectives
The goal of this project is to build a modern identity management solution that is robust, secure and scalable. Users and groups should be handled centrally, instead of existing as separate local entities on each host. Policies and rules dictate access and authentication. This should not come at a cost for the end-user, who should still be able to use their machines without significant hindrance. 

## Method


## Target Audience
This project is for anyone who wants to learn about FreeIPA, and implementing it in a multi-client, multi-user environment. This repo is also part of a larger project aimed at people interested in learning about IT-infrastructure and production, and building such an environment from scratch.


## Document Status
This project is finished, but is part of a larger project that is currently unfinished. This project can be followed indepently of previous projects. 

## Disclaimer
This project is intended for lab-environments, with a focus on learning, testing, and experimentation. Although we keep security in mind, we can't promise that this implementation is suitable for production.

## Scope and Limitations
- Many FreeIPA features have not been covered here, so think of this as an introductory project.
- This project uses a one-server setup. In a real setting, we want redundancy in cases where the FreeIPA-server goes down. This is outside the scope of this project but can be accomplished using [replication](https://www.freeipa.org/page/V4/Replica_Setup). 
- Instructions may become outdated as software updates; always verify with the official documentation.

## Environment
- FreeIPA version 4.12.2
- Proxmox VE (9.1.1)
- Rocky Linux (10.1)
- Ansible (core 2.16.14)

## Acknowledgments
We would like to thank <a href=https://github.com/rafaelurrutiasilva>Rafael Urrutia</a> for his continuous support and guidance.

## Implementation

### Prepare a new VM
It is suggested to run the FreeIPA-server on its own VM. Other VMs, like *mgmt-01*, will depend on IAM functionality. Keeping seperate VMs avoids circular dependency, and makes recovery easier. 

#### Clone rocky-base
Create a new clone from the *rocky-base* template. Give it the name *ipa-01*, a VM ID and static IPv4- and IPv6-addresses.

#### Add new VM in Ansible
On the *mgmt-01* VM, add the new *ipa-01*-host to the Ansible *hosts.ini* file. We'll put it under the *[rocky]* category and also create a new *[ipaservers]* category for it. 

Copy over your public SSH-key to the IPA VM. Confirm that it's working with: 
```
ansible ipaservers -m ping
```

#### Install packages

On the *ipa-01* VM, install the ipa-server and ipa-server-dns packages:
```
sudo dnf install ipa-server ipa-server-dns
```

### DNS

FreeIPA needs DNS to work correctly, which includes; forward and reverse DNS resolution, stable FQDNs, correct SRV records and time sync resolvable by name. 

#### Decide on a domain structure

We will not use our organisations domain (smhi.se) for this lab, since we don't want service-record collisions, or other nasty side-effects. An alternative to this is using a subdomain, such as lab.smhi.se. A second alternative is using an internal domain, which we will use instead. FreeIPA can be installed with its own integrated DNS, which we will use as our internal DNS server for the lab environment. We'll still need to access our organisations DNS server, and the FreeIPA DNS can be configured to forward traffic to it.

Choose a name for the private lab domain, like *.plab.internal*. The VMs will now belong to this domain, and have names like *ipa-01.plab.internal*, *mgmt-01.plab.internal*, and so on. 

Go to the Proxmox web GUI, and make the following changes on every VM: <br>
Cloud-Init > DNS domain: *plab.internal* <br>
Cloud-Init > DNS servers: *ip-address of ipa-01* <br>

#### Update /etc/hosts, hostnames and /etc/resolv.conf on VMs

Go into the shell of each VM, edit the /etc/hosts-file to reflect this new naming convention. For example, on *metrics-01*:
```
# The following lines are desirable for IPv4 capable hosts
127.0.0.1 metrics-01.plab.internal metrics-01
127.0.0.1 localhost.localdomain localhost
127.0.0.1 localhost4.localdomain4 localhost4

# The following lines are desirable for IPv6 capable hosts
::1 metrics-01.plab.internal metrics-01
::1 localhost.localdomain localhost
::1 localhost6.localdomain6 localhost6
```

Change the hostname:
```
hostnamectl set-hostname metrics-01.plab.internal
```

Verify name change with:
```
hostname -f
```

Also edit */etc/resolv.conf* to point to the new domain:
```
search plab.internal
nameserver xxx.xxx.xxx.xxx #ip-address of ipa-01
```

for *ipa-01*:
```
search plab.internal
nameserver 127.0.0.1
nameserver ::1
```

Note that changing hostnames will have some initial consequences for the environment. SSH will warn about not recognizing the hostname of hosts, this can be fixed with `ssh-keygen -R user@host`. 

### FreeIPA installation

#### Install ipa-server

Run:
```
sudo ipa-server-install
```

This will take you through a configuration process. Say 'yes' to internal DNS. You could most likely accept the default values given for hostname, domain and realm. 

The installation should take a minute or two. Be observent here, as it will warn you if it runs into any issues. For us, it gave the following warning:
*"Warning: IPA was unable to sync time with chrony!"*

The installation was still successful. This just means we should fix time-sync to ensure proper Kerberos functionality.

### Configure DNS

For DNS to work properly in this arrangement, the integrated DNS must forward traffic to the company DNS. 

Show if DNS is running:
```
ipa dnsconfig-show
```

Add forwarder:
```
ipa dnsconfig-mod --forwarder=xxx.xxx.xxx.xxx
```

Set forward policy to *only*:
```
ipa dnsconfig-mod --forward-policy=only
```

Restart internet domain name server:
```
sudo systemctl restart named
```

If you run into DNS issues, a potential fix is turning off dnssec validation, which doesn't need to be enabled for an internal environment like this. Open up *ipa-options-ext.conf*:
```
sudo vi /etc/named/ipa-options-ext.conf
```

Set dnssec-validation to no:
```
/* dnssec-enable is obsolete and 'yes' by default */
dnssec-validation no;
```

Restart DNS:
```
sudo systemctl restart named
```

### Chrony/NTP

#### Check Chrony

Check if an NTP server is being used by chrony: 
```
chronyc tracking 
```

If the reference ID shows *00000000 ()* then there is no server in use. 

#### Update chrony.conf with Ansible

We haven't yet added NTP on our VMs. To do this, we'll use Ansible. Go into the *mgmt* VM, and create a new playbook:
```yaml
---
- name: Configure chrony NTP server
  hosts: rocky
  become: yes

  tasks:
    - name: Ensure NTP server is configured
      ansible.builtin.lineinfile:
        path: /etc/chrony.conf
        regexp: '^pool 2\.rocky\.pool\.ntp\.org iburst'
        line: 'server xxx.xxx.xxx.xxx iburst'
        state: present
      when: ansible_distribution == "Rocky"

    - name: Restart chrony
      ansible.builtin.service:
        name: chronyd
        state: restarted
```

This playbook will rewrite the *chrony.conf* file to use our NTP server, and then restart chronyd.

Note that this playook is intended for Rocky Linux hosts. The chrony configuration will initially point to a rocky-specific ntp pool, which we'll use a regular expression to identify and remove. Also, the location of this config-file is not necessarily the same for other Linux-distributions. 

Verify chrony/NTP with:
```
chronyc tracking
```

Verify time synchronization with:
```
timedatectl
```

### Firewall configuration

FreeIPA uses a couple of different protocols. The following ports must be allowed through the firewall for full functionality:
- TCP ports: 80, 433 (HTTP/HTTPS), 389, 636 (LDAP/LDAPS), 88, 464 (Kerberos), 53 (DNS)
- UDP Ports: 88, 464 (Kerberos), 123 (NTP)

We'll configure the firewall on the Proxmox server to work with these protocols.
We've already made rules for HTTP,HTTPS, DNS and NTP. These rules doesn't need to be adjusted. Though the server will need some additional DNS rules. 

Since we are using an internal DNS, and we want to forward traffic to our organisations DNS, we want to replace the ip-address of the *dns* IP Set.

#### FreeIPA-server firewall rules
Create a new security-group and call it *freeipa-server*.
Add the following rules:

<pre>
Direction: in
Action: ACCEPT
Enable: yes
Protocol: tcp
Dest. port: 88
Log level: info
</pre>

<pre>
Direction: in
Action: ACCEPT
Enable: yes
Protocol: tcp
Dest. port: 636
Log level: info
</pre>

<pre>
Direction: in
Action: ACCEPT
Enable: yes
Protocol: tcp
Dest. port: 389
Log level: info
</pre>

<pre>
Direction: in
Action: ACCEPT
Enable: yes
Protocol: tcp
Dest. port: 464
Log level: info
</pre>

<pre>
Direction: in
Action: ACCEPT
Enable: yes
Macro: DNS
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Macro: DNS
Log level: info
</pre>

#### FreeIPA-client firewall rules
Create a new IP set, give it the name *ipa-servers*
Add the address of the *ipa-01* VM.

Create a new security-group, call it *freeipa-client*.
Add the following rules:
<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: tcp
Destination: +ipa-servers
Dest. port: 88
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: tcp
Destination: +ipa-servers
Dest. port: 636
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: tcp
Destination: +ipa-servers
Dest. port: 389
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: tcp
Destination: +ipa-servers
Dest. port: 464
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: udp
Destination: +ipa-servers
Dest. port: 88
Log level: info
</pre>

<pre>
Direction: out
Action: ACCEPT
Enable: yes
Protocol: udp
Destination: +ipa-servers
Dest. port: 464
Log level: info
</pre>

#### Apply Security groups
On the VM-level, apply the *freeipa-server* security-group to the *ipa-01* VM. For the rest of the VMs, apply the *freeipa-client* security group.

I also suggest adding a comment to the two new security-groups, stating that they do not include rules for HTTP, HTTPS and NTP. The name of the *dns* IP set can also be changed to *dns-internal*. Update the DNS security group with the new name. These changes help reflect the current state of the project. 

### Set up FreeIPA-clients with Ansible

When installing the IPA-clients, it is a good idea to use Ansible. This ensures correct state on all hosts and prevents configuration-drift. Ansible can also install FreeIPA-servers, which is recommended when setting up multiple servers.

#### Install ansible-freeipa

On the *mgmt* VM, run:
```
sudo dnf install ansible-freeipa
```

<a href=https://github.com/freeipa/ansible-freeipa>ansible-freeipa</a> is a collection for Ansible, containing roles, playbooks and modules to perform FreeIPA-related tasks. With this, we'll install and configure the FreeIPA-client on our hosts.

This package will install to */usr/share/ansible/collections/ansible_collections/freeipa/ansible_freeipa/*. For convenience's sake, we'll make a symbolic link in */opt/ansible*.

Link to playbook:
```
ln -s /usr/share/ansible/collections/ansible_collections/freeipa/ansible_freeipa/playbooks/install-client.yml /opt/ansible/playbooks/install_ipa_client.yml
```

We only need <a href=https://github.com/freeipa/ansible-freeipa/blob/master/playbooks/install-client.yml>this playbook</a>.

The tasks for this installation process are found under <a href=https://github.com/freeipa/ansible-freeipa/blob/master/roles/ipaclient/tasks/install.yml>roles/ipaclient/install.yml</a>.

If we like to keep the relevant ansible-files in one and the same folder, that is /opt/ansible, we can specifically set the paths in the ansible.cfg file. Beside the playbook, we'll make a link for the /roles/ipaclient directory:
```
ln -s /usr/share/ansible/collections/ansible_collections/freeipa/ansible_freeipa/roles/ipaclient/ /opt/ansible/roles/ipaclient/
```

check current roles path:
```
ansible-config dump | grep -i roles_path
```

To change roles path, open *ansible.cfg* and add:
```
roles_path = /opt/ansible/roles
```


#### Prepare hosts.ini

On the *mgmt* VM, open */inventory/hosts.ini* and add:

```ini
[ipaservers]
ipa-01

[ipaclients]
metrics-01
app-01
mgmt-01
```

#### Ansible Vault

FreeIPA-clients need the admin password and directory manager password from the server to be configured. We want to keep these secret when working with Ansible, and not store them in plaintext. Ansible vault encrypts data at rest, and allows playbooks to safely access these. 

Create a vault.yml file for secrets:
```
ansible-vault create /opt/ansible/inventory/group_vars/ipaclients/vault.yml
```

Add these lines:
```yaml
---
vault_ipaadmin_password: password
vault_ipadm_password: password
```

#### vars

The */group_vars* and */host_vars* folders placed in inventory is meant for *vars.yml*-files associated to groups and hosts. Here, we can define attributes that apply only to the groups and hosts we want. 

Create a new folder in /group_vars for ipaclients:
```
mkdir -p /opt/ansible/inventory/group_vars/ipaclients
```

Create a vars.yml file:
```
ansible-vault create /opt/ansible/inventory/group_vars/ipaclients/vars.yml
```

vars.yml will contain plaintext variables. It will also reference the passwords stored in the vault by their variable names. 

```yaml
---
ipaadmin_password: {{ vault_ipaadmin_password }}
ipadm_password: {{ vault_ipadm_password }}
ipa_server: ipa-01.plab.internal
ipaserver_domain: lab.internal
ipaserver_realm: LAB.INTERNAL
ipaclient_force_join: yes
ipaclient_mkhomedir: yes
```

These variables are from the FreeIPA-collection and their descriptions are found in the 
 <a href=https://github.com/freeipa/ansible-freeipa/blob/master/roles/ipaclient/README.md#variables>documentation.</a> 

#### Install FreeIPA clients

Run the playbook:
```
ansible-playbook /opt/ansible/playbooks/install_ipa_client.yml
```

#### Create a vault-password file

The vault password can be stored in a password file. That will skip the ansible-vault password prompt when running playbooks. 

Create the file:
```
echo 'password' > ./opt/ansible/.vault_pass
```

Set permissions for file:
```
chmod 660 .vault_pass
```

In *ansible.cfg*, add the following line:
```
vault_password_file = .vault_pass
```

Remember to not commit this file in git, use *gitignore*! 

It's also a good idea to clear the command history afterwards:
```
history -c
```

### Verify that FreeIPA works so far

FreeIPA consists of many different components, and confirming that they all seem healthy can save you a lot of headache later.

#### IPA Server

Log into the *ipa-01* VM.

Check that all services are running:
```
sudo ipactl status
```

Check server registration:
```
kinit admin
ipa server-find
```

Verify host enrolment:
```
ipa host-find
```

#### DNS

Forward lookup:
```
dig ipa-01.plab.internal
```

Reverse lookup:
```
dig -x xxx.xxx.xxx.xxx
```

SRV records lookup:
```
dig _kerberos._tcp.plab.internal SRV
dig _ldap._tcp.plab.internal SRV
```

#### Kerberos

From any IPA-client, try to obtain a Kerberos ticket:
```
kinit admin
klist
```

#### SSSD

From any IPA-client, check if SSSD is running:
```
systemctl status sssd
```

check that the admin IPA-user is visible:
```
getent passwd admin
```

#### TLS

From client to server, using curl:
```
curl https://ipa-01.lab.internal/ipa/json
```

If no certificate errors show up, and the return output is either JSON or 401/unauthorized, then CA trust is established. 

### IPA users

Currently, each VM has a couple of users: rocky, jonatan and Filip. These are local Linux-users, and they're not bound to FreeIPA. jonatan and Filip are human users, and we want FreeIPA counterparts. It wouldn't fit in the FreeIPA paradigm to have both, so the local Linux users will have to be removed. rocky is a special case, since it's the Rocky Linux cloud-init user, and not tied to any human user. System users should not be managed by IPA, so we'll ignore rocky.

#### Remove local users

In case you have important files owned by these users, either consider migrating the user ID to the IPA-user, or changing file ownership. 

Delete corresponding Linux-users on all VMs (*remove=yes* deletes user directories):
```
ansible all -b -m user -a "name=jonatan state=absent remove=yes"
```

#### Add IPA users

When working with FreeIPA, you can either use its Web UI, or the CLI. The *ipa* command will offer the same functionality as the Web UI. We'll be using the CLI. 

Start by initializing the admin account:
```
kinit admin
```

Add user:
```
ipa user-add jonatan --homedir=/home/jonatan --shell=/bin/bash --password
```

Make sure to give the user a password, since this will function as the user's initial authentication. 

Verify that FreeIPA can see the user:
```
getent passwd jonatan
```

Initialize the user:
```
kinit jonatan 
```

Attempting this, FreeIPA will likely tell you that your password have expired and prompt you to enter a new password. You can pick the same password again! 

Confirm password expiration:
```
ipa user-show jonatan --all | grep -i "user password expiration:"
```

Format is YYYY-MM-DD-HH:MM:SS

Now you can try to log in on any of the VMs as that user:
```
ssh jonatan@hostname
```

When logging in for the first time, that users home directory should be automatically created.

Create at least one more user, it will be useful for testing later. 


### IPA groups

We'll create two groups for this lab, one for admins and one for regular users. Note that the group *admins* already exists by default (this is where the *admin* user is). We'll leave this group alone, and call our other admin-group something like *sysadmins*. 

Initialize admin:
```
kinit admin
```

Create a group for admins:
```
ipa group-add sysadmins
```

Create a group for regular users:
```
ipa group-add users
```

Add user to the group:
```
ipa group-add-member sysadmins --users=jonatan
```

Verify:
```
ipa group-show sysadmins
```

Add the second user to the user group:
```
ipa group-add-member users --users=Filip
```

### HBAC

Host-based access control (HBAC) determines which users/groups are allowed to use which hosts/services.
We will use the following arrangement:
- Sysadmins have full access to all VMs.
- Users have limited access to metrics and app.

#### host groups

Like with users, hosts are best managed on a group basis. We'll make host-groups that reflect the category of VMs we have.

Create host groups:
```
ipa hostgroup-add ipa
ipa hostgroup-add mgmt
ipa hostgroup-add metrics
ipa hostgroup-add app
```

Tie VMs to groups:
```
ipa hostgroup-add-member ipa --hosts=ipa-01.plab.internal
ipa hostgroup-add-member mgmt --hosts=mgmt-01.plab.internal
ipa hostgroup-add-member metrics --hosts=metrics-01.plab.internal
ipa hostgroup-add-member app --hosts=app-01.plab.pinternal
```

#### HBAC rules

Create a new rule:
```
ipa hbacrule-add sysadmin-access --servicecat=all
```

the `--servicecat=all` option adds all available services to this rule. 

Add user group to rule:
```
ipa hbacrule-add-user sysadmin-access --groups=sysadmins
```

Add host groups to rule:
```
ipa hbacrule-add-host sysadmin-access --hostgroups=ipa
ipa hbacrule-add-host sysadmin-access --hostgroups=mgmt
ipa hbacrule-add-host sysadmin-access --hostgroups=metrics
ipa hbacrule-add-host sysadmin-access --hostgroups=app
```

Enable the rule:
```
ipa hbacrule-enable sysadmin-access
```

Verify:
```
ipa hbacrule-show sysadmin-access
```

HBAC starts with one default rule, *allow_all*, which is equivocal to having HBAC turned off. This rule will have to be disabled:
```
ipa hbacrule-disable allow_all
```

Keep in mind that disabling this rule enforces a *deny all* behaviour. Other rules should be in place beforehand. 

Test the rule:
```
ipa hbactest --host mgmt-01.plab.internal --service sudo --user jonatan
ipa hbactest --host mgmt-01.plab.internal --service sudo --user filip
```

Create a rule for users:
```
ipa hbacrule-add user-access
ipa hbacrule-add-user user-access --groups=users
ipa hbacrule-add-host user-access --hostgroups=metrics
ipa hbacrule-add-host user-access --hostgroups=app
ipa hbacrule-add-service user-access --hbacsvcs=sshd
ipa hbacrule-add-service user-access --hbacsvcs=login
ipa hbacrule-enable user-access
```

Test:
```
ipa hbactest --host app-01.plab.internal --service sshd --user filip
ipa hbactest --host mgmt-01.plab.internal --service sshd --user filip
```

### sudo rules

Sudo rules can be used to define which users have access to which commands, and more. Again, we'll keep it simple and grant sysadmins complete sudo access. 

Create a new sudo-rule:
```
ipa sudorule-add full-sudo --hostcat=all --runasusercat=all --runasgroupcat=all --cmdcat=all
```

Add rule to group:
```
ipa sudorule-add-user full-sudo --group sysadmins
```

Add the *!authenticate* option to the rule, which disable password-checks:
```
ipa sudorule-add-option full-sudo --sudooption='!authenticate'
```

This option is necessary for running ansible plays with remote root privileges. 

Verify:
```
sudo -l
```

### SSH key management

At this point, Ansible won't work with our new IPA-users. This is because we haven't generated and shared SSH-keys for these users yet. We could do this like we've done in previous labs, but an alternative solution is using FreeIPA to store and manage our public keys for us. This makes management of SSH keys much more convenient, and scales better. It also reinforces the FreeIPA-server as our central identity and access management solution, and single source of truth. 

Go on the *mgmt* VM and change ownership and permissions of ansible files:
```
sudo chown -R root:sysadmins /opt/ansible
sudo chmod -R g+rw opt/ansible
```

Make sure you're logged in with your *sysadmins*-user and generate a new SSH-key:
```
cd ~
ssh-keygen
```

Tie this key to the IPA user:
```
ipa user-mod jonatan --sshpubkey="$(cat /home/jonatan/.ssh/id_ed25519.pub)"
```

Confirm:
```
ipa user-show jonatan --all
```

Verify that Ansible can reach remote hosts:
```
ansible all -m ping
```

### Break-glass admin

It's a good idea to have a local user account with root access on standby. A *break-glass* user, which exists in case of emergencies where FreeIPA becomes unavailable.

We've removed local users, with the exception of *rocky*, our Rocky Linux cloud-init user account. rocky is not suitable for the role of a *break-glass* user, nor does it fit with an IAM solution.

#### Create break-glass user
We'll use Ansible to create a local user on all VMs. This ensures that the user is properly mirrored across systems.

Create a new playbook:
```yaml
---
- name: Create breakglass user
  hosts: rocky
  become: true
  vars:
    breakglass_user: breakglass
    
  tasks:
    - name: Create breakglass user
      ansible.builtin.user:
        name: "{{ breakglass_user }}"
        password: "{{ vault_breakglass_password_hash }}"
        comment: "Emergency admin"
        shell: /bin/bash
        create_home: true
        password_lock: false

    - name: Set permissions
      ansible.builtin.copy:
        dest: /etc/sudoers.d/"{{ breakglass_user }}"
        content: |
          "{{ breakglass_user }}" ALL=(ALL) ALL
        owner: root
        group: root
        mode: '0440'
        validate: /usr/sbin/visudo -cf %s

    - name: Verify sudo works
      ansible.builtin.command: sudo -l -U "{{ breakglass_user }}"
      changed_when: false
```

Before running this playbook, we must first create an encrypted password. Ansible will use the <a href=https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/user_module.html#parameter-password>hashed value of the password</a>.

Create the password/hash:
```
openssl passwd -6
```

Copy the resulting hash, and open the *vault.yml* file:
```
ansible-vault edit /opt/ansible/inventory/group_vars/all/vault.yml
```

Add a new line:
```
vault_breakglass_password_hash: $6$H3URzXnmGEKLbL.H$VqAwromUzOw...
```

Save and exit, then clear history.

Run playbook:
```
ansible-playbook ./playbooks/create_breakglass_user.yml
```

Confirm that the user is created and works as intended:
```
su breakglass
sudo id
```

### Subordinate UID/GID

Subordinate user and group IDs (subuid/subguid) are a way to assign users the root ID When running rootless containers with Podman. Local users are automatically given a subordinate ID range, while IPA-users are not. User namespaces should not be handled by FreeIPA.

Add users to /etc/subuid and /etc/subgid for the VMs running podman and buildah:
```
ansible app -b -m ansible.builtin.lineinfile -a 'path=/etc/subuid line="jonatan:2222:65536" create=yes'
ansible app -b -m ansible.builtin.lineinfile -a 'path=/etc/subgid line="jonatan:2222:65536" create=yes'
```

## Conclusion
With this project, we have only managed to scratch the surface of FreeIPA and IAM. Major components of FreeIPA have been left out, like the dogtag certificate system and the web UI. We haven't added things like web application authentication, SELinux user maps or Kerberos ticket policies. Although we kept things relatively simple with users, groups, HBAC and sudo-rules, the granularity and flexibility of these systems have not gone unnoticed. In a real enterprise setting, we would likely need to define more user-groups with varying sets of restrictions imposed on them, we would likely follow IAM standards such as NIST SP 800-63, and much more. 

## References
- [FreeIPA](https://www.freeipa.org/)
- [SMHI](https://www.smhi.se/en/about-smhi)
- [Ansible FreeIPA collection](https://github.com/freeipa/ansible-freeipa)
- [Ansible builtin module - Password parameter](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/user_module.html#parameter-password)
- [FreeIPA Replica Setup](https://www.freeipa.org/page/V4/Replica_Setup)

**Other parts in our project:**
- Part 1 - [Installing Proxmox on an Asus PN64](https://github.com/rafaelurrutiasilva/Proxmox_on_Nuc)
- Part 2 - [Rocky Linux Golden Image](https://github.com/Filipanderssondev/Rocky_Linux_OS_Base_for_VMs)
- Part 3 - [Ansible on management VM](https://github.com/JonatanHogild/Ansible_on_management_vm)
- Part 4 - [Container stack deployment with Ansible](https://github.com/Filipanderssondev/Container_Stack_Deployment_With_Ansible/)


