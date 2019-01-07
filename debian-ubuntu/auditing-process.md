# Simple auditing process for Debian / Ubuntu
This is a process for performing a basic audit of your system configuration. The commands here are tailored towards Debian / Ubuntu. Altough with a few tweaks you can do the exact same thing for RedHat based distributions.

## System / software out of date

### Check version of Linux
Check whether you are running an old version of Linux. Compare the running version with the latest version.

```
uname -a
lsb_release -a
```

### Check for missing updates and patches.
This a bit more labour intensive for people who don't have professional security tools that can automate patch checking. Ideally, you can keep your system up-to-date with `apt-get update` but as a point of thoroughness you would check all software that's installed.
```
dpkg -l
```

### Check if SSH is out of date
Pay particular attention to versoin of SSH you are using. Older versions can be vulnerable.
```
ssh -version
ssh -V
dpkg -l | grep ssh
```

### Check for shellshock
Old systems will be vulnerable to shellshock. There is an easy way to check your system:

```
x='() { :;}; echo "***WARNING: BASH VULNERABLE TO SHELLSHOCK"' bash -c :
```



# System Authentication and Access
## Check for weak passwords
You should know if you are using weak passwords on your own system. There are a couple checks you can do to get an indication on someone else's system. If you have root privileges then you could look into the shadow file for signs of weak hashes, and you can also attempt to crack the hashes. If you don't have a shadow file then you problems! Passwords shouldn't be reused across systems either: e.g. login passwords should be different for each machine.

Performing a `grep` for `pass` will show you what password restrictions are in place on your system. Essentially, this is the password policy. Weak requirements allow for weak passwords. Ideally, you should be creating passwords in a passoword manager with sufficient length and complexity.

```
grep -i "pass" /etc/login.defs
```

For example, the minimum length:
```
grep PASS_MIN_LEN /etc/login.defs
```



### Accounts with unnecessary shell access / logins
Linux system accounts don't require have valid shells, so these can be removed from the accounts in `/etc/passwd`. Look to see if any of the system accounts have (e.g.) `/bin/sh` or `/bin/bash`.

Also investigate whether any role accounts have interactive logins. They don't require this.

```
cat /etc/passwd
```

### The .rhosts file is being used for authentication
```
ls /etc/.rhosts
```

Ideally this file won't exist. You can double check the file doesn't exist anywhere, by using: `find / -name ".rhosts" -ls`

### Sudo file is poorly configured
First check the access on the file. Only the root account should be able to read and write to it.
e.g.  `-r--r----- 1 root root`

```
ls -l /etc/sudoers
```

Also check whether the sudoers file contains any daft rules, e.g. `username  ALL=(ALL) NOPASSWD: ALL`
```
grep NOPASSWD /etc/sudoers
```

That would allow `username` to perform any sudo command without a password. You have to inspect the sudoers file manually:
```
cat /etc/sudoers
```

### Check for rlogin/rexec/rsh
Check if your system uses rlogin/rexec/rsh. Probably not going to be seen on modern systems. I believe that commands like `rsh` now point to SSH instead.

```
netstat -apn 
ps -aef | grep -i rsh
```



## Check the configuration of the SSH Server daemon
There are a number of configuration issues pertaining to SSH. Given that SSH is often used for remote access to Linux systems, it is wise to put considerable time into getting this part right.

So either you want to `cat` (or `less`) the file and look at the configuration at once, or grep for each parameter one at a time.

```
cat /etc/ssh/sshd_config
```

### Check if root user can login
Check if your system allows the root account to login remotely. All systems have a root user which means that it is an obvious username to try when attacking a system remotely. It is harder to guess the name of a low privileged user and then break into that account and escalate up to root.

In particular, check the ssh configuration.
```
grep PermitRootLogin /etc/ssh/sshd_config
```

Bad: `PermitRootLogin yes`.


And potentially check all places that root can log into:
```
cat /etc/securetty
```

### Maximum number of retries for authentication
The default should be set to 6, check that the line is not commented out, and check that the number of retries is not too high.

```
grep MaxAuthTries /etc/ssh/sshd_config
```

### Don't allow empty passwords
It would be a bad idea to allow SSH with empty passwords. Make sure the following is **not** set to `yes`.

```
grep PermitEmptyPasswords /etc/ssh/sshd_config
```

### Remove password based authentication
Ideally, you should turn password authentication off. This will require authentication with a private key (much safer), but the public key has to be on the target system **before** you turn off password authentication.

```
grep PasswordAuthentication /etc/ssh/sshd_config
```

### Ensure that the login time is not too long

```
grep LoginGraceTime /etc/ssh/sshd_config
```

### Check / change SSH port number
Personally, I think you should change your SSH port to something else. Something high and 'random'. It is only a minor point of additional security but it should help to avoid some of the opportunistic scanners. 

```
grep Port /etc/ssh/sshd_config
```

### Check the SSH protocol version
By default your SSH daemon should be set to accept only protocol version 2. This wouldn't be the case if the system is ancient, but take a double check:

```
grep Protocol /etc/ssh/sshd_config
```

### Check the minimum key length
Check the length of bits required for keys (this is for RSA keys, I believe). By default the figure is 768 which is far too low. I'd not recommend 1024 either, but to push for 2048.

```
grep ServerKeyBits/etc/ssh/sshd_config
```

### SSH `AcceptEnv' Enabled
There have been vulnerabilities found in the past where server being logged into had accepted environment variables from the accessing system.

```
grep AcceptEnv /etc/ssh/sshd_config
```

A bad configuration is `AcceptEnv LANG LC_*`.

### Check which users can read the  Authorized Keys file
This is the file that's checked when logging in via SSH. It shouldn't be accessible to everyone. It can be found under `/home/<username>/.ssh`.

```
find / -name authorized_keys -ls
```

### Check if the private keys are protected by a password
Your target system may or may not be used to further SSH into another box. For many use-cases you won't want to be doing this. However, for the few cases that you are using a jump-box, then you should ideally be protecting all SSH private keys with a password. You can find all private key files then take a look to see if they are encrypted (ideally they are encrypted with a good password too!).

```
grep -l 'BEGIN [RD]SA PRIVATE KEY' /home/<username>/.ssh/*
```

Make sure to pay particular attention to keys for the root user.

```
grep -l 'BEGIN [RD]SA PRIVATE KEY' /root/.ssh/*
grep -l 'ENCRYPTED' /root/.ssh/*
```



# Poor file permissions
It would take a long time to check all file permissions and try to determine if they allow for access. 

### Check for files that have SUID / GUID bits set
First, it is worth checking which files have either SUID/GUID bits set.

```
find / -type f -a \( -perm -u+s -o -perm -g+s \) -ls
```

### Check if start files are world-writeable
Check if the startup file is writeable (init.d)

```
ls -l /etc/init.d/
```

### Check who can access cron
Check if the cron files can be access by other users. They should only be accessed by root.

```
ls -l /etc/ | grep "cron"
ls -l /etc/cron*
```

### Look for world-accessible files
Also, check if there are any files / directories that are World-Writable (and to a lesser extent World-Readable). This is mostly a  worry when the device is connected to the Internet, has multiple users, or in outside chance that the device is somehow compromised and an attacker is able to login with a low privileged account. If attacker has the root password then they can do a lot of damage anyway.
```
find / -type f -perm -002 -ls
```

Smart attackers may try to alter log files too. Take a double check that log files are not world-writeable, e.g.
```
ls -la /var/log
```

### Root user's home folder is accesible by low privileged users
Double check the home directory of the root user is not accessible by low privileged users.
```
ls -al /root
ls -al /home
```

### Check for nouser/ nogroup
Check if there are any files or directories owned by 'nouser' / 'nogroup'.

```
find / -nouser -ls
find / -nogroup -ls
```


## Network Config
### Check firewall configuration
Check if firewall rules are (both ipv4 and v6) configured. This requires some understanding of the context of the system, but if there are no rules set then there is a problem. If you see that random ports are allowed then you might have some old rules that are no longer valid.

Look at the firewall rules with:
```
iptables -L
ip6tables -L
```

### Unnecessary services
Check for unnecessary network services running
```
netstat -apn
```


