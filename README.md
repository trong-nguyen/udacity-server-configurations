# Linux Server Configurations

## Create a Linux instance using AWS Lightsail

- Connect using your own shell. Download the pem key from [Lightsail](https://lightsail.aws.amazon.com/ls/webapp/account/keys)
```shell
# chmod to 400
chmod 400 path/to/your/lightsail/key.pem

# connect
ssh -i path/to/your/lightsail/key.pem username@public_ip

# copy public SSH key to the Lightsail instance
local@ssh-keygen # create a public key on the local machine
# copy content of the .pub key to clipboard and echo it to the remote authorized_keys file
echo "rsa-....(public key content)" >> ~/.ssh/authorized_keys
```



## Initial configurations and network security

```shell
# update package lists
sudo apt-get update

# upgrade installed packages and dependencies
sudo apt-get dist-upgrade

# remove no longer required libraries
sudo apt autoremove

# open port 2200, get prepared for ssh port change (from 22 to 2200)
sudo ufw allow 2200
# enable and reload
sudo ufw enable
sudo ufw reload # (opt)
sudo ufw status # (opt) check status

# change ssh port 22 -> 2200 (look for a line starting with "Port 22")
sudo nano /etc/ssh/sshd_config
sudo service sshd restart # restart ssh service

# one last step: on the Lightsail web panel, under networking, add a custom port to be opened at 2200
```

- Setup firewall to enable ports
```shell
sudo ufw allow in 2200
sudo ufw allow in 80
sudo ufw allow in 123
sudo ufw reload # take effect after reloading
```

## Setup `grader` user

```shell
sudo adduser grader
# sudo passwd -e grader # (opt) expire grader password immediately, enforce password change - Udacity
su - grader # login to grader

# add sudo permission to grader
# either edit the /etc/sudoers
sudo visudo # and copy the line
    root ALL=(ALL:ALL) ALL
    to grader ALL=(ALL:ALL) ALL

# or (recommended by Ubuntu) create a new file in the /etc/sudoers.d/ such as /etc/sudoers.d/grader-user with the following line
    grader ALL=(ALL) NOPASSWD:ALL

# generate ssh key following instructions
ssh-keygen
```

## Deploy app


## Resources
- Uncomplicated FireWall [ufw](https://www.linux.com/learn/introduction-uncomplicated-firewall-ufw)
