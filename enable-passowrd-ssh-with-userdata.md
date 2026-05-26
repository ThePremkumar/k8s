# Steps to Enable Password Authentication and Set Password for First Login on EC2

---

## Instructions & Configuration

Below is the shell script and the required steps to enable SSH password authentication and configure credentials.

```bash
#!/bin/bash
sed 's/PasswordAuthentication no/PasswordAuthentication yes/' -i /etc/ssh/sshd_config
systemctl restart sshd
service sshd restart

#TODO: replace bob with your desired username
useradd bob
# TODO: replace password123 with desired password and change bob to your username chosen in useradd 
echo "password123" | passwd --stdin bob
echo "password123" | passwd --stdin root
```
