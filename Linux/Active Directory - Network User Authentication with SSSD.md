# SSSD and Active Directory

This section describes the use of sssd to authenticate user logins against an Active Directory via using sssd’s “ad” provider. At the end, Active Directory users will be able to login on the host using their AD credentials. Group membership will also be maintained.

## Prerequisites, Assumptions, and Requirements
 - This guide does not explain Active Directory, how it works, how to set one up, or how to maintain it.
 - This guide assumes that a working Active Directory domain is already configured and you have access to the credentials to join a machine to that domain.
 - The domain controller is acting as an authoritative DNS server for the domain.
 - The domain controller is the primary DNS resolver (check with systemd-resolve --status)
 - System time is correct and in sync, maintained via a service like chrony or ntp
 - The domain used in this example is ad1.example.com .

## Software Installation
Install the following packages: sudo apt install sssd-ad sssd-tools realmd adcli

## Join the domain
We will use the realm command, from the realmd package, to join the domain and create the sssd configuration.
Let’s verify the domain is discoverable via DNS: sudo realm -v discover ad1.example.com

This performs several checks and determines the best software stack to use with sssd. sssd can install the missing packages via packagekit, but we installed them already previously.
Now let’s join the domain: sudo realm join ad1.example.com

That was quite uneventful. If you want to see what it was doing, pass the -v option: sudo realm join -v ad1.example.com

By default, realm will use the Administrator account of the domain to request the join. If you need to use another account, pass it to the tool with the -U option.
Another popular way of joining a domain is using an OTP, or One Time Password, token. For that, use the --one-time-password option.

## SSSD Configuration
The realm tool already took care of creating an sssd configuration, adding the pam and nss modules, and starting the necessary services.
Let’s take a look at /etc/sssd/sssd.conf:

Note: Something very important to remember is that this file must have permissions 0600 and ownership root:root, or else sssd won’t start!

Let’s highlight a few things from this config:
- cache_credentials: this allows logins when the AD server is unreachable
- home directory: it’s by default /home/<user>@<domain>. For example, the AD user john will have a home directory of /home/john@ad1.example.com
- use_fully_qualified_names: users will be of the form user@domain, not just user. This should only be changed if you are certain no other domains will ever join the AD forest, via one of the several possible trust relationships

## Automatic home directory creation
What the realm tool didn’t do for us is setup pam_mkhomedir, so that network users can get a home directory when they login. This remaining step can be done by running the following command: sudo pam-auth-update --enable mkhomedir

## Checks
You should now be able to fetch information about AD users. In this example, John Smith is an AD user: getent passwd john@ad1.example.com
Let’s see his groups: groups john@ad1.example.com

Note: If you just changed the group membership of a user, it may be a while before sssd notices due to caching.

Finally, how about we try a login: sudo login ad-client login: john@ad1.example.com

Notice how the home directory was automatically created. You can also use ssh, but note that the command will look a bit funny because of the multiple @ signs: ssh john@ad1.example.com@10.51.0.11

Note: In the ssh example, public key authentication was used, so no password was required. Remember that ssh password authentication is by default disabled in /etc/ssh/sshd_config

## Kerberos Tickets
If you install krb5-user, your AD users will also get a kerberos ticket upon logging in: john@ad1.example.com@ad-client:~$ klist

Note: realm also configured /etc/krb5.conf for you, so there should be no further configuration prompts when installing krb5-user

Let’s test with smbclient using kerberos authentication to list he shares of the domain controller: john@ad1.example.com@ad-client:~$  smbclient -k -L server1.ad1.example.com
Notice how we now have a ticket for the cifs service, which was used for the share list above: john@ad1.example.com@ad-client:~$ klist

# References
 - https://ubuntu.com/server/docs/service-sssd-ad
