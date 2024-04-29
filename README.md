This is small Guide to setup a secure (relative, because as soon as you expose you to the outside, there might be some attacks). Kudos to PitbullCH, a fellow node-op on a different project who shared most of the VPS setup information (This is a subset of the instructions). This guide is written to be used for Linux-OS on both, host machine and remote server (Debian).

login to your newly setup server via ssh (most provider have ssh-server installed on your machine)

```ssh root@xxx.xxx.xxx.xxx```

### Change password

On the remote server, change the password set by either you or the provider

```passwd root```

to check if new password login works, open a 2nd terminal on login (do not close first terminal as it is a backup connection)

### Add new user

On the remote server, for security reasons add a non-root user

```adduser your_user```

On the remote server, add this user to the sudoers and allow to access system logs

````
usermod -aG sudo your_user
usermod -aG systemd-journal your_user
````

log out and login with new user 

```ssh your_user@xxx.xxx.xxx.xxx```

On the remote server, check which groups you belong to

```groups```

### Set up ssh

set up ssh-keys for more secure logins on your local machine. If you already have some ssh-keys, do store the new ones under different names as otherwise you will loose your keys and can not connect to servers you previously used. If those are your first ssh-keys, use default directory and name and set a password.

```ssh-keygen```

copy the public key to your remote server 

```ssh-copy-id your_user@xxx.xxx.xxx.xxx```

in a new terminal on your local machine, login via ssh (your ssh-key password is needed)

```ssh your_user@xxx.xxx.xxx.xxx```

on your remote server: to secure the ssh logins, we need to change some parts in the sshd_config file on your remote machine

````
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.myback
sudo nano /etc/ssh/sshd_config
````

on your remote server: The following fields need to be uncommented and or changed to (we set a new ssh port as most attempts to login via ssh will try port 22 first. the new port should be a high number port not used by the application you will run like 45453. Note: firewall such as ufw should not be active here. If it is, first allow the new ssh port before you you logout of your remote server): 
````
Port 45453
PermitRootLogin no
MaxAuthTries 3
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
PermitEmptyPasswords no
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
````

on your remote server: Test the new ssh configuration

```sudo sshd -t```

on your remote server: restart the ssh service

```sudo service ssh restart```

logout and login again 

```ssh your_user@xxx.xxx.xxx.xxx -p 45789```

on your remote server: if something went wrong, undo the changes by coping back the backup-file

````
sudo cp /etc/ssh/sshd_config.myback /etc/ssh/sshd_config
sudo service ssh restart
````
### Update system

on your remote server: update the system

````
sudo apt-get update
sudo apt-get ugrade
sudo reboot
````

### Install NTP

on your remote server: Some blockchain nodes are sensitive to time drifts, so NTP could be installed

````
sudo apt-get update
sudo apt-get install ntp ntpdate
sudo service ntp stop
sudo ntpdate pool.ntp.org
sudo service ntp start
sudo systemctl status ntp
````

### Install Fail2Ban

on your remote server: Install Fail2Ban to block repeating incomming connection attempts

````
sudo apt-get update
sudo apt-get install fail2ban
sudo cp /etc/fail2ban/jail.conf
/etc/fail2ban/jail.local
````

on your remote server: backup the file

```sudo cp /etc/fail2ban/jail.local /etc/fail2ban/jail.myback```

on your remote server: edit the file (as we changed default ssh port)

```sudo nano /etc/fail2ban/jail.local```

and change the 3 ssh ports from "port ssh" to "port 45453". It will look sth like: 

````
port = 45453
logpath = %(sshd_log)s
backend = %(sshd_backend)s
[dropbear]
Port = 45453
logpath = %(dropbear_log)s
backend = %(dropbear_backend)s
[selinux-ssh]
port = 45453
logpath = %(auditd_log)s
````

on your remote sever: start Fail2Ban

````
sudo systemctl enable fail2ban
sudo service fail2ban start
sudo systemctl status fail2ban
````

on your remote server: if something went wrong, undo everything by

````
sudo cp /etc/fail2ban/jail.myback /etc/fail2ban/jail.local
sudo service fail2ban stop
sudo systemctl status fail2ban
````
### Set up firewall

on your remote server: update firewall (make sure to allow the new ssh port 45453. 10000 is used for webmin later (optional). also replace VNC by your VNC port
````
sudo ufw disable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 45453
sudo ufw allow VNC (optional)
sudo ufw allow 10000 (optional)
sudo ufw logging on
sudo ufw enable
sudo ufw status
````

### Install Webmin (OPTIONAL)

on your remote server: Install Webmin for remote surveilance. Do not forget to activate 2FA
````
sudo apt-get update
sudo apt-get install gpg-agent apt-transport-https software-properties-common
wget -q -O- http://www.webmin.com/jcameron-key.asc | sudo apt-key add
sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib
sudo apt-get update
sudo apt-get install webmin
sudo systemctl status webmin
````

on your local machine: open a browser and connect to: 

```http://your_server_name:10000```

login with youre non-root user. Note, you can setup webmin to work with https if you want

### Install Docker / Docker-compose

on your remote machine: install docker and docker-compose
````
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo groupadd docker
sudo usermod -aG docker your_user

sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
````

check the installations. Note, no need to use sudo infront of docker as we added our user the specific group

````
docker run hello-world
docker–compose --version
````

### Install Bashtop

on your remote server: install Bashtop for system resource checking

````
wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add
echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list
sudo apt-get update
sudo apt-get install bashtop
bashtop
````

### Server maintenance

Periodically maintain your remote server

````
sudo apt update
sudo apt upgrade
sudo apt-get update
sudo apt-get upgrade
````

### Useful commands for beginner node-ops

#### ls

ls is used to list content of a certain directory

| Command | Effect |
| --- | --- |
| ls | list all visible content in the current directory |
| ls /home/alice/Documents | list all visible content at /home/alice/Documents |
| ls -a | lists also hidden content |
| ls -l | lists additional information such as owner of files |

#### cd

"cd" is used to navigate through your file system.

| Command | Effect |
| --- | --- |
| cd | navigate to your home directory |
| cd ~ | navigate to your home directory |
| cd sample_dir | relative navigation from you current directory to "sample_dir" |
| cd /usr/local | navigation independent of your current directory |

#### pwd

pwd returns you the absolute path of your current location

#### mkdir

mkdir creates a new directory 
| Command | Effect |
| --- | --- |
| mkdir new_dir | creates new_dir in the current directory |
| mkdir /home/alice/Documents/new_dir | creates a new_dir at /home/alice/Documents/new_dir |

#### cp

cp is used to copy contend from one location to another. You can think of it as copy and past.

| Command | Effect |
| --- | --- |
| cp /home/alice/sample.txt /home/alice/Documents | copy the file sample.txt to Documents |
| cp /home/alice/samle_dir /home/alice/Documents | copy the directory samle_dir to Documents |

#### mv

mv is used to move files from one location to another. You can think of it as cut and past.

| Command | Effect |
| --- | --- |
| mv /home/alice/sample.txt /home/alice/Documents | move the file sample.txt to Documents |
| mv /home/alice/samle_dir /home/alice/Documents | move the directory samle_dir to Documents |
| mv /home/alice/old_filename.txt /home/alice/new_filename.txt | rename the file old_filename.txt to new_filename.txt |

#### rm

rm is used to remove files and directories

| Command | Effect |
| --- | --- |
| rm sample.txt | remove the file sample.txt |
| rm -r samle_dir | remove the directory samle_dir |

Hint: if you do not have permission to delete a file, either change your permission or use sudo

#### chmod

chmod is used to change file permission.

#### find

find is used to search your filesystem for a certain file

| Command | Effect |
| --- | --- |
| sudo find / -name sample.txt | this will use root-privileges to search for a file named sample.txt through your whole filesystem |

### Useful tools for beginner node-ops

#### Text editors

One of the simplest text editors is nano. You can use nano to create a new or edit and existing file. You can select different endings such as .txt, .yml, .yaml, .json, .py, etc.
The command below will create a new file called sample.txt in your current directory, or will open that file if it already exists in your current directory.

````
nano sample.txt
````
To exit the editor press ctr+x and chose to either save or discard changes.

#### netstat

Use netstat to check if you have services listening on your ports. 

````
sudo netstat -tunlp | grep LISTEN
````

You can install netstat by running:

````
sudo apt-get update
sudo apt install net-tools
````

#### tmux

tmux is a terminal multiplexer. You can use it to run tasks in the background. When you run binaries in your terminal, they will abort as soon as you close the terminal. You can use tmux to run those binaries in the background.

| Command | Effect |
| --- | --- |
| tmux new -s session_1 | starts a detachable bash session named session_1 |
| ctrl+b "up"-key | allows you to scroll within the detachable session |
| esc | exits the scroll-mode |
| ctrl+b d | detaches the session without terminating the process that currently run |
| tmux a -t session_1 | reattaches the session named session_1 |
| ctrl+b "up"-key | allows you to scroll within the detachable session |

You can install tmux by running:

````
sudo apt-get update
sudo apt-get install tmux
````
