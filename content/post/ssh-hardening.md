+++
date = "2016-03-23T00:00:00+01:00"
title = "Hardening SSH of your VPS"
Description = "SSH hardening for simple setups"

translationurl = "/br/post/ssh-hardening/"

tags = [
	"SSH",
    "Hardening"
]
+++

## TL;DR
VPS - Virtual Private Servers - usually comes with a ton of default settings. Here I describe how to properly secure SSH for a very simple setup, like a private VPS where I am the only user and I have full control of server and client.

## Intro
One of the things of having a VPS is to have the freedom to do whatever you want on a system that is up all the time [(or most of the time)](https://www.ovh.com/us/dedicated-cloud/discover/sla.xml). I use OVH for reasons that escape me at the moment. In the end, it doesn't matter. Wherever your VPS is, be it in Amazon AWS, Microsoft Azure, Linode or your [favourite "underground"/privacy conscious provider accepting bitcoins](http://libertyvps.net/), it will not use by default the best configuration for a security minded administrator.

In this day and age, SSH is the de facto standard for remote access. Good old times of telnet and 10Mbps hubs, where you could sniff everything, are over. However, doing cryptography is always a challenge, and SSH is no different.

The main purpose of using SSH and a cryptographic tunnel between two computers is to protect the content being sent through the encrypted channel - also many more security features like authentication and message integrity. If this channel is compromised for any reason, bad things can happen.

There are [several](http://www.coresecurity.com/content/ssh-insertion-attack) [types of](https://blog.sucuri.net/2013/07/ssh-brute-force-the-10-year-old-attack-that-still-persists.html) [attacks](https://sites.google.com/site/clickdeathsquad/Home/cds-ssh-mitmdowngrade) that focus on the cryptography side of SSH and not to the protocol itself. Just google "SSH attacks".

Throughout this post I will show you the sources of information I collected to build my SSH configuration. I will not draw images of how SSH works, key exchange, MACs, whatever because there are already a lot of sites out there with amazing explanations. I will also not go through long explanations about cryptography- Even though it's interesting, it's not the point for this post.

Shout out to [stribika's blog post](https://stribika.github.io/2015/01/04/secure-secure-shell.html) that inspired me to do this. Although I am not doing this to protect me from a particular three letter agency who shall not be named. Security is freely available and I want to use the best. End.

## The Start

First of all, you need to understand one fundamental piece of information regarding SSH that I overlooked for a LONG time. Connecting one computer with another involves TWO computers. It is *not enough* to harden only your SSH server. Please refer to the [CVE-0216-0777 and CVE-0216-0778](https://www.digitalocean.com/community/tutorials/how-to-fix-openssh-s-client-bug-cve-0216-0777-and-cve-0216-0778-by-disabling-useroaming) regarding UseRoaming and also this amazing piece of work by [Filippo Valsorda](https://github.com/FiloSottile/whosthere) where he can identify who you are by the SSH key you use.

We will start with the server configuration and move on to the client configuration. *DON'T SKIP THE CLIENT CONFIGURATION* like I did for many years of my life.

## The Server

You are given a new VPS. Hurray! Do you even know what keys your SSH server is using and cryptographic options that created them? Most of them rely on automatic key generation from the openssh server package during VPS automatic provisioning, which is fair enough. The fact is that I don't trust them enough (for SSH key creation methods, not for having the ability to access all my data and scrape my server's memory and steal all my keys).

There are *horror stories* like [this one](http://wiki.hetzner.de/index.php/Ed25519/en) where Hetzner, a VPS provider, reused the same SSH server key for a number of their clients due to a configuration error.

`Due to an error in the installation software introduced on April 10th, 2015, the Ed25519 SSH host keys (/etc/ssh/ssh_host_ed25519_key) on our standard images were no longer automatically regenerated. This resulted in identical Ed25519 SSH host keys for each affected OS image.` -- Hetzner

Remember. I am the only user of this system. I decide which encryption level I want and I don't care about backward compatibility and old SSH clients. My objective is to use the highest encryption possible, avoid downgrade attacks, make it difficult for adversaries to unencrypt my encrypted traffic. I generate and protect my own keys.

1. Recreate the SSH server key  
	There is no reason for your server to offer multiple keys for server authentication. The `ssh-keygen` tool offers a number of options for the type of key to generate. My option is to always use [ed25519](https://ed25519.cr.yp.to/) even though RSA with a bigger key is equally secure. Mine will be called `/etc/ssh/ssh_server_ed25519`.  
	
	```
	ssh-keygen -t ed25519 -f /etc/ssh/ssh_server_ed25519 -o -a 128
	```

	This key cannot have a passphrase. This is how SSH works. You reboot your server, SSH comes up, and it works. Make sure that the mode for the new file is 600 (-rw-------).  
	Notice the `-o -a 128` parameters. This is the number of rounds used to derive the key. The default is 16. Our powerful computers are capable of doing the math in about 2 seconds. There is a good explanation for this [here](https://martin.kleppmann.com/2013/05/24/improving-security-of-ssh-private-keys.html)  
	To avoid misconfiguration, make sure you delete the old server keys that are already available.  

	Also, note down the public key fingerprint of the newly generated key. The output is something like this:

	```
	The key fingerprint is:
	ed:9a:3d:e6:dd:61:4d:99:cd:68:09:6c:b0:af:e2:4e example.com
	```

	You can verify later with the following command:  
	```
	ssh-keygen -lf /etc/ssh/ssh_server_ed25519.pub
	```

2. Reconfigure `/etc/ssh/sshd_config`  
	The bulk of the configuration and explanation is here. Here is a dump of my `sshd_config` and we will go step by step. Some of the lines are default in SSH but I want to make sure they are there and be crystal clear of their meaning.
	```
	Port 22
	Protocol 2
	HostKey /etc/ssh/ssh_server_ed25519
	AllowGroups ssh-allowed
	ClientAliveInterval 300
	ClientAliveCountMax 0
	KexAlgorithms curve25519-sha256@libssh.org
	Ciphers chacha20-poly1305@openssh.com
	MACs hmac-sha2-512-etm@openssh.com
	UsePrivilegeSeparation yes
	SyslogFacility AUTH
	LogLevel VERBOSE
	LoginGraceTime 120
	PermitRootLogin no
	StrictModes yes
	PubkeyAuthentication yes
	IgnoreRhosts yes
	HostbasedAuthentication no
	PermitEmptyPasswords no
	ChallengeResponseAuthentication no
	PasswordAuthentication no
	X11Forwarding no
	PrintMotd no
	PrintLastLog yes
	TCPKeepAlive yes
	UseLogin no
	PermitTunnel no
	AcceptEnv LANG LC_*
	Subsystem sftp /usr/lib/openssh/sftp-server
	UsePAM no
	```

	Here is the explanation of the important pieces:

	2.1) `Port 22`  
		Only allow connections on port 22 in all server's interfaces. Changing this value for "security" reasons is security by obscurity and it's ridiculous in my opinion. You'd rather use a firewall rule to filter inbound source IP addresses instead.

	2.2) `Protocol 2`  
		Because protocol 1 is fundamentally broken.

	2.3) `HostKey /etc/ssh/ssh_server_ed25519`  
		The new server key I just created.

	2.4) `AllowGroups ssh-allowed`  
		Only users from linux group `ssh-allowed` are allowed to login through SSH. This is to protect myself from my own errors, like creating a new temporary user and affecting the security of my server.

	2.5) `ClientAliveInterval 300` and `ClientAliveCountMax 0`  
		This is user inactivity timeout. This way enforces timeouts on the server side.

	2.6) `KexAlgorithms curve25519-sha256@libssh.org`  
		There is no reason to use multiple key exchanges algorithms. Using ECDH over Curve25519 with SHA2 is perfectly ok and I don't want to bother with moduli bit sizes for Diffie-Hellman group exchanges.

	2.7) `Ciphers chacha20-poly1305@openssh.com`  
		This is the symmetric key used to finally encrypt the content going through between client and server. This cipher offers the best available, avoids traffic analysis and so on.

	2.8) `MACs hmac-sha2-512-etm@openssh.com`  
		For the integrity of the communication, this algorithm provides Encrypt-then-MAC with the biggest key size available.

	2.9) `UsePrivilegeSeparation yes`  
		Do not run the ssh daemon with a privileged user account. This creates unprivileged child processes to deal with network traffic. If the server is compromised through the ssh daemon, the attacker will gain unprivileged rights and will need to work harder to escalate privileges.

	2.10) `LogLevel VERBOSE`  
		We have plenty of storage, why not logging every single piece of information regarding one of the most important services in my VPS.

	2.11) `PermitRootLogin no`  
		This is the default configuration but some VPS give you remote root access by default, thus changing this to another value. You don't want to use root for SSH. Create your own unprivileged user to connect remotely and then escalate privilege if and when necessary.

	2.12) `StrictModes yes`  
		This protects me from misconfiguration. It will check if the key file permissions and other things are properly configured.

	2.13) `PubkeyAuthentication yes`  
		Allow public key authentication. This is fundamental when disallowing password authentication.

	2.14) `IgnoreRhosts yes` and `HostbasedAuthentication no`  
		This is default but I just want to make sure.

	2.15) `PasswordAuthentication no` and `PermitEmptyPasswords no`  
		I want to logon using my user key and not my Linux user password.

	2.16) `ChallengeResponseAuthentication no`  
		This is a tricky one and I use them interchangeably. Even the sshd-config man page is difficult to understand. If I only want to authenticate using my user key, I set this to `no`. Later on I discuss two-factor authentication, where this needs to be changed to `yes`.

	2.17) `X11Forwarding no`  
		I'm not using X11, the default is `no` already but I want to make sure.

	2.18) `UseLogin no`  
		This is the default and I want to make sure `login(1)` is not used for user authentication.
	
	2.19) `PermitTunnel no`  
		This is the default but just making sure I'm only using SSH to connect to my VPS, nothing else.

	2.20) `UsePAM no`  
		This is similar to ChallengeResponseAuthentication. If I'm only using my user key to authentication, this can be `no`. If I'm using something fancier, this is `yes`.

## The Client

The client configuration somewhat follows the server configuration. You generate strong keys, you define cryptographic algorithms for everything, disable some stuff, done.

But the key point here is to use different keys for different servers. In the past, I had one key and added that to authorized_keys in every single server I had access to. In the end, you have no idea where your key is, which systems you have access to, etc. The risk here is that if your private key is compromised, lost, stolen, copied by mistake on dropbox/github/USB dongle, it will be very difficult to ascertain the damage.

Therefore, I propose to use one key pair per each server I manage. Yes, it overloads administrative tasks, it's a burden, but hey, we are talking about security best practices and I don't have access to that many servers anyway. I'm talking about my own VPS. So why not creating one separate and specific key to access it? This is also valid to services like Github and anything else that uses SSH.

The file for client configuration, from a user's perspective (not root's perspective), is `~/.ssh/config`. For the sake of `StrictModes`, everything under my `.ssh` folder has mode `0600`. 

1. Configure default behaviour in `~/.ssh/config` file:
	This needs to follow what we configured for our VPS. I want to tell my ssh client to only use that specific crypto algorithm, with that key exchange method with that specific cipher. The entry for `Host *` provides global defaults for all hosts, so I don't need to repeat them in every single `Host` entry:

	```
	Host *
	    KexAlgorithms curve25519-sha256@libssh.org
	    Ciphers chacha20-poly1305@openssh.com
	    MACs hmac-sha2-512-etm@openssh.com
	    ForwardAgent no
	    UseRoaming no
	```

	Notice two extra lines in this configuration:  
	1.1) `ForwardAgent no`
		This is the default configuration but I want to make sure that if I ever use the ssh-agent to manage my private keys, I don't want it to forward it to remote machines.

	1.2) `UseRoaming no`  
		The infamous undocumented "feature" and could expose user's keys to attackers on a MITM scenario.

	This will be applied to all servers you connect from now on. If a server needs a different parameter, do it specifically for that Host. You will *quickly realize* that this may break connectivity to some servers, *including Github*. Below you see the example for Github and the specifics for its configuration.

2. Create a key for every single server/system you have access to:
	In this example, I create one to use with Github. Don't forget to add a complex passphrase:
	```
	ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "user@computer" -o -a 128
	``

	The comment here with option `-C` is important so you easily identify where that key belongs to. Imagine a server where you connect with 10 different machines - which public key belongs to which machine? Do this with the comment.

3. Configure `~/.ssh/config` for this specific service:
	```
	Host github.com
	    Hostname ssh.github.com
	    Port 443
	    MACs hmac-sha2-512
	    IdentityFile ~/.ssh/id_ed25519_github
	```

	Here is [Github's help](https://help.github.com/articles/using-ssh-over-the-https-port/) on how to do it.  
	For your VPS, the process is the same. Create a new key pair, create the `Host` block in your `~/.ssh/config` file, add your public key to your user's `authorized_keys` file on the server side and you are done. Here is my entry for my own VPS:
	```
	Host example.com
		ServerAliveInterval 30
		StrictHostKeyChecking yes
		IdentityFile ~/.ssh/id_ed25519_example.com
	```
	In this example, I add the option `StrictHostKeyChecking` to make sure that if something happens to the server's key - the fingerprint changed, I'm under MITM attack, whatever - I am blocked from connecting. This prevents the automatic update of my `known_hosts` file, which could be catastrophic.

4. Add your key to ssh-agent:
	Seriously, you don't want to keep typing in your super strong passphrase everytime you logon to your VPS. It makes it harder for attackers if your local machine is compromised and all your keystrokes are recorded. So, imagine this. Even through the attacker has a copy of your key, sees everything you type but he is not able to logon to your VPS from any other machine other than yours. Surely he could potentially add a new key to your authorized_keys on your VPS but that's a flag right there. This is where two-factor authentication comes in.

## Two-Factor Authentication

In my opinion, authenticating with a key that I own and protect is sufficient. Even if my private key is compromised, it is encrypted with a complex password that is difficult to brute-force. My SSH server does not allow anything other than a key to authenticate, so my local Linux user credentials is safe. I can also block source IPs temporarily if they brute-force my login using tools like [fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page).

But, there are cases where I want to use a second factor of authentication. This can be achieved with `google-authenticator`, where you use your mobile device to generate OTP - One Time Passwords - that are necessary to authenticate alongside your key. So even though the attacker has your key and your passphrase, he cannot login because he doesn't have your phone.

In the next few post I will discuss ssh-agent and two-factor authentication further.