# Title Page
<img width="100" alt="MyLogo" src="https://github.com/rafaelurrutiasilva/images/blob/main/logos/MyLogo_2.png" align=left><br>
<br>
Titel på dokumentet<br>
Författare<br>
Publiceringsdatum<br>

<br>

## Abstract
Kort sammanfattning av dokumentet

## Table of Contents

1. [Introduction](#introduction)
2. [Goals and Objectives](#goals-and-objectives)
3. [Method](#method)
4. [Target Audience](#target-audience)
5. [Document Status](#document-status)
6. [Disclaimer](#disclaimer)
7. [Scope and Limitations](#scope-and-limitations)
8. [Environment](#environment)
9. [Acknowledgments](#acknowledgments)
10. [References](#references)
11. [Conclusion](#conclusion)

## Introduction
Inledning
Bakgrund och syfte. Eventuell översiktbild här.

## Goals and Objectives
Mål och syften

## Method
### 3.1 Prepare a new VM
It is strongly suggested to run the FreeIPA-server on its own VM. Other VMs, like mgmt-01, will depend on IAM functionality. Keeping seperate VMs avoids circular dependency. Misconfiguration in this lab could lead to IPA policy denying your login, effectively locking you out of your own identity system. This solution prevents the risk of such lock-out accidents.

#### 3.1.1 Clone rocky-base
Create a new clone from the *rocky-base* template. Give it the name *ipa-01*, a VM ID and static IPv4- and IPv6-addresses.

#### 3.1.2 Add new VM in Ansible
On the *mgmt-01* VM, add the new *ipa-01*-host to the Ansible *hosts.ini* file. We'll put it under the *[rocky]* category and also create a new *[ipaservers]* category for it. 

Copy over the public SSH-key of the *mgmt* VM to the IPA VM. Confirm that it's working with: 
```
ansible ipaservers -m ping
```

#### 3.1.3 Install packages

On the *ipa-01* VM, install the ipa-server and ipa-server-dns packages:
```
sudo dnf install ipa-server ipa-server-dns
```

### 3.2 DNS

FreeIPA needs DNS to work correctly, which includes; forward and reverse DNS resolution, stable FQDNs, correct SRV records and time sync resolvable by name. 

FreeIPA can be configured to use its own integrated DNS, which we will use for this lab. We'll still need to access our organisations DNS server, and the FreeIPA DNS will forward traffic to it.

#### 3.2.1 Decide on a domain

choose a name for the private lab domain, like *.plab.internal*. The VMs will now belong to this domain, and have names like *ipa-01.plab.internal*, *mgmt-01.plab.internal*, and so on. <br>

Go to the Proxmox web GUI, and make the following changes on every VM: <br>
Cloud-Init > DNS domain: *plab.internal* <br>
Cloud-Init > DNS servers: *ip-address of ipa-01* <br>

#### 3.2.2 Update /etc/hosts, hostnames and /etc/resolv.conf on VMs

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

### 3.3 FreeIPA installation

#### 3.3.1. Install ipa-server

Run:
```
sudo ipa-server-install
```

This will take you through a configuration process. Say 'yes' to internal DNS. You could most likely accept the default values given for hostname, domain and realm. 

The installation should take a minute or two. Be observent here, as it will warn you if it runs into any issues. For us, it gave the following warning:
*"Warning: IPA was unable to sync time with chrony!"*

The installation was still successful. This just means we should fix time-sync to ensure proper Kerberos functionality.

### 3.3.2 Configure DNS

For DNS to work properly in this arrangement, the integrated DNS must forward traffic to the company DNS. 

Show if DNS is running:
```
ipa dnsconfig-show
```

Add forwarder:
```
ipa dnsconfig-mod --forwarder=xxx.xxx.xxx.xxx
```

set forward policy to *only*:
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

### 3.4 Chrony/NTP

#### 3.4.1 Check Chrony

Check if an NTP server is being used by chrony: 
```
chronyc tracking 
```

If the reference ID shows *00000000 ()* then there is no server in use. 

#### 3.4.2 Update chrony.conf with Ansible

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

### 3.4 Firewall configuration

FreeIPA uses a couple of different protocols. The following ports must be allowed through the firewall for full functionality:
- TCP ports: 80, 433 (HTTP/HTTPS), 389, 636 (LDAP/LDAPS), 88, 464 (Kerberos), 53 (DNS)
- UDP Ports: 88, 464 (Kerberos), 123 (NTP)

We'll configure the firewall on the Proxmox server to work with these protocols.
We've already made rules for HTTP,HTTPS, DNS and NTP. These rules doesn't need to be adjusted. Though the server will need some additional DNS rules. 

Since we are using an internal DNS, and we want to forward traffic to our organisations DNS, we want to replace the ip-address of the *dns* IP Set.

#### 3.4.1 FreeIPA-server firewall rules
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

#### 3.4.2 FreeIPA-client firewall rules
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

#### 3.4.3 Apply Security groups
On the VM-level, apply the *freeipa-server* security-group to the *ipa-01* VM. For the rest of the VMs, apply the *freeipa-client* security group.

I also suggest adding a comment to the two new security-groups, stating that they do not include rules for HTTP, HTTPS and NTP. The name of the *dns* IP set can also be changed to *dns-internal*. Update the DNS security group with the new name. These changes help reflect the current state of the project. 

### 3.5 Set up FreeIPA-clients with Ansible

When installing the IPA-clients, it is a good idea to use Ansible. This ensures correct state on all hosts and prevents configuration-drift. 

#### 3.5.1 Install ansible-freeipa

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


#### 3.5.2 Prepare hosts.ini

On the *mgmt* VM, edit the Ansible */inventory/hosts.ini* and add:

```ini
[ipaservers]
ipa-01

[ipaclients]
metrics-01
app-01
mgmt-01
```

#### 3.5.3 Ansible Vault

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

#### 3.5.4 vars

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

#### 3.5.5  Install FreeIPA clients

Run the playbook:
```
ansible-playbook /opt/ansible/playbooks/install_ipa_client.yml
```

#### 3.5.6 Create a vault-password file

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

### 3.6 Verify that FreeIPA works so far

FreeIPA consists of many different components, and confirming that they all seem healthy can save you a lot of headache later.

#### 3.6.1 IPA Server

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

#### 3.6.2 DNS

Forward lookup:
```
dig ipa-01.company.com
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

#### 3.6.3 Kerberos

From any IPA-client, try to obtain a Kerberos ticket:
```
kinit admin
klist
```

#### 3.6.4 SSSD

From any IPA-client, check if SSSD is running:
```
systemctl status sssd
```

check if the admin IPA-user is visible:
```
getent passwd admin
```

#### 3.6.5 TLS

From client to server, using curl:
```
curl https://ipa-01.lab.internal/ipa/json
```

If no certificate errors show up, and the return output is either JSON or 401/unauthorized, then CA trust is established. 

### 3.7 IPA users

Currently, each VM has a couple of users: rocky, jonatan and Filip. These are local Linux-users, and they're not bound to FreeIPA. jonatan and Filip are human users, and we want FreeIPA counterparts. It wouldn't fit in the FreeIPA paradigm to have both, so the local Linux users will have to be removed. rocky is a special case, since it's the Rocky Linux cloud-init user, and not tied to any human user. System users should preferably not be managed by IPA, so we'll ignore rocky (for now).

#### 3.7.1 Remove local users

In case you have important files owned by these users, either consider migrating the user settings to FreeIPA (UID/GID), or using *chown* later. 

Delete corresponding Linux-users on all VMs (*remove=yes* deletes user directories):
```
ansible all -b -m user -a "name=jonatan state=absent remove=yes"
```
#### 3.7.2 Add IPA users

When working with FreeIPA, you can either use its Web UI, or the CLI. The *ipa* command will offer the same functionality as the Web UI. We'll be using the CLI. 

Start by initializing the admin account:
```
kinit admin
```

Add user:
```
ipa user-add jonatan --uid=1001 --gidnumber=1001 --homedir=/home/jonatan --shell=/bin/bash --password
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


### 3.8 IPA groups

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

### 3.9 HBAC

Host-based access control (HBAC) determines which users/groups are allowed to use which hosts/services. We will use the following arrangement:
- Sysadmins have full access to all VMs.
- Users have limited access to metrics and app.

#### 3.9.1 host groups

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

#### 3.9.2 HBAC rules

HBAC-rules state which users have access to what hosts, and which services on those hosts. For this lab, things will be kept simple. A regular user will have access to only SSH and login services. Sysadmins will have access to everything. 

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

### 3.10 sudo rules

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

This is necessary for running ansible plays with *-become*. 

Verify:
```
sudo -l
```

<pre>
User jonatan may run the following commands on ipa-01:
    (ALL : ALL) NOPASSWD: ALL
</pre>

### 3.11 SSH

At this point, Ansible won't work with our new IPA-users. This is because we haven't generated and shared SSH-keys for these users yet. We could do this like we've done in previous labs, but an alternative solution is using FreeIPA to store and manage our public keys for us. This is not only more convenient, it further consolidates our centralized identity management solution. 

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

### 3.12 Break-glass admin

It's a good idea to have a local user account with root access on standby. A *break glass* user, which exists in case of emergencies where FreeIPA breaks or otherwise becomes unavailable. 

We've removed local users, with the exception of *rocky*, our Rocky Linux cloud-init user account. rocky is not suitable for the role of a *break glass* user, for many reasons. In fact, rocky does not fit into our design at all.

password-lock rocky:
```
ansible all -b -m user -a "name=rocky password_lock=true"
```

This is a nice, not-so drastic method of retiring rocky. Just make sure that *rocky* does not have SSH access to any of the hosts using public key authentication. 

#### 3.12.1 Create break-glass user
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

## Target Audience
Målgrupp

## Document Status
Dokumentstatus (om det finns relevant information om dokumentets status, till exempel utkast, slutfört, etc.). Tex:
> [!NOTE]  
> My work here is not finished yet. I need, among other things, to supplement with instructions on how each component should be configured to work together as well supplement with an overview image that explains how the whole thing works.


## Disclaimer
Ansvarsfriskrivning. Tex:
> [!CAUTION]
> This is intended for learning, testing, and experimentation. The emphasis is not on security or creating an operational environment suitable for production.

## Scope and Limitations
Omfattning och begränsningar

## Environment
Miljö som användes

## Acknowledgments
Tack och erkännanden. Tex:
Big thanks to all the people involved in the material I refer to in my links! I would also like to express gratitude to everyone out there, including my colleagues and friends, who are creating things that help and inspire us to continue learning and exploring this never-ending world of computer technology.

## References
Referenser (om det behövs)

## Conclusion
Slutsats
