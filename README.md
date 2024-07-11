# Configure Firewall with iptables

## Objective
Working with my Ubuntu server, the goal was to create a firewall that locked down the server and only allowed traffic in that was intended by me, but could be done from virtually any machine. This was done on an SSH hardened Ubuntu server ([see SSH hardening project](https://github.com/naomi-kerr/SSH-Hardening)). The goal was to only allow local traffic within the system, incoming traffic that was approved, and log dropped traffic for troubleshooting.
### Skills Learned
* iptables rules
* iptables routing
* port blocking
### Tools Used
* iptables
* SSH
* iptables-save
## Steps
1. Allow loopback, established, and related traffic
2. Allow SSH traffic over designated port
3. Allow basic outgoing traffic (http/s, dns, etc.)
4. Log traffic dropped in designated log file
5. Change INPUT & OUTPUT policy to DROP
6. Save iptables rules
## Process
*Note1: The last rule in your iptables needs to be your log rule. The traffic is run through rules starting with the first rule and continuing through each rule until the traffic is accepted. With the policy set to DROP, the iptables will log all traffic that was not accepted and then drop it after it is logged. If placed anywhere else, the rule will log traffic that may be accepted later on.  To see rules and their order numbered, use:*
```
sudo iptables -vnL --line-numbers | less
```

*Note2: It is recommended not to do this unless physical access to the machine is available. If the SSH rule is entered incorrectly and the policy is set it drop, it will lock the client out of the server entirely. Changing or resetting the rules cannot be done remotely if this happens.*

*Note3: Ensure that the policy drop is the last step in this process. If enabled too early it, like above, can lock the client out of the server entirely. *

*Note4: The rules created will need to be saved using* `iptables-save`. *If the server goes down or is rebooted, all created rules will be lost and the process will need to be repeated.*
### Allow Loopback, Established, & Related Traffic
Enabling loopback traffic ensures that anything connecting over the NIC within the host is allowed to do so. Anything that needs connection via localhost will require this rule in order for the connection to be established.  
```
sudo iptables -I INPUT -i lo -m comment --comment "Allow incoming loopback traffic" -j ACCEPT
```

Next, any traffic that is sent using an established or a related connection should be allowed. This means that after a connection is created (via SSH on this machine), other types of incoming traffic are accepted from the connected machine. 

`sudo iptables -A INPUT -m conntrack -ctstate ESTABLISHED,RELATED -m comment --comment "Allow incoming established and related traffic" -j ACCEPT`  

The same thing is done for loopback traffic in the OUTPUT table to ensure that traffic communicating via localhost is able to do so. 
```
sudo iptables -I OUTPUT -o lo -m comment --comment "Allow outgoing loopback traffic"
```

Lastly, allow established and related outgoing traffic. This will allow the server to send communication to the client or other machines it has established a connection with.
```
sudo iptables -A OUTPUT -m conntrack -ctstate ESTABLISHED,RELATED -m comment --comment "Allow outgoing established and related traffic"
```

### Allow SSH Traffic Over Designated Port
Because this machine is a headless server without any other services running, the only traffic to be concerned about at the moment is SSH. This will allow us to remotely access the server. The loopback, established, and related rules created above allow the client to interact with the machine without issue. Reaching out to other devices on the network and the internet will be enabled with the outgoing traffic rules next. 
```
sudo iptables -A INPUT -p tcp --dport 57625 -j ACCEPT
```

### Outgoing Traffic Rules
In order to use web browsers and initiate contact with other machines on the network, various protocols need to be enabled for outgoing traffic. In this case, it is a minimal set of rules designed for web browsing and FTP/S. Review the comments in the code for what service each rule is enabling. 

```
sudo iptables -A OUTPUT -p tcp --dport 20 -m comment --comment "Allow FTP outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p tcp --dport 21 -m comment --comment "Allow FTPS outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p tcp --dport 22 -m comment --comment "Allow FTPS outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p tcp --dport 53 -m comment --comment "Allow DNS over TCP outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p udp --dport 53 -m comment --comment "Allow DNS over UDP outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p tcp --dport 80 -m comment --comment "Allow HTTP traffic outgoing" -j ACCEPT

sudo iptables -A OUTPUT -p tcp --dport 443 -m comment --comment "Allow HTTPS outgoing" -j ACCEPT
```

### Create Log File & Log Rules
With the firewall being so locked down, it's important to log dropped traffic. Logging traffic like this is extremely helpful when troubleshooting network connection issues on the server (particularly when creating new rules to allow more incoming traffic). 

First, create the log file at `/var/log/iptables.log` using `touch`. 
```
sudo touch /var/log/iptables.log
``` 

Next, open `/etc/rsyslog.conf` or `/etc/syslog.conf` (whichever is on your system) in your text editor of choice and add the following:
```
#iptables kernal logging to /var/log/iptables.log
kern.*  /var/log/iptables.log
```

Restart your syslog to initiate the changes
```
sudo service rsyslog restart
```

Lastly, add the logging rules for both the INPUT and OUTPUT tables. 
```
sudo iptables -A INPUT -m comment --comment "Log dropped INPUT traffic" -j LOG --log-prefix "***Dropped INPUT Traffic***" --log-level 6        #Logging Rule

sudo iptables -A OUTPUT -m comment --comment "Log dropped OUTPUT traffic" -j LOG --log-prefix "***Dropped OUTPUT Traffic***" --log-level 6
```

To check that the log is working, use `tail` to print the last 10 lines of the log. If there is no traffic going to the server, attempt to SSH into the wrong port. This will populate the log with a failed SSH attempt.
```
sudo tail /var/log/iptables.log
```

All of the rules are set up and the last step is to change the Policy to DROP. 
### Change Policy
Setting the policy to DROP means that any traffic which isn't accepted by one of the rules created will not be able to connect to the server. The iptables default policy for each table is ACCEPT, which allows any and all traffic to make it to the server, leaving it vulnerable. To set the policy to drop for the INPUT and OUTPUT tables, use the following commands: 
```
sudo iptables -P INPUT DROP

sudo iptables -P OUTPUT DROP
```

### Test SSH and Save
If physical access to the server is possible, use `exit` to leave the server. Then, SSH back into the machine using the port established in your rules. If physical access is not possible and the rules were created over SSH, use another client/machine to test SSH access. If SSH is successful, then the rules are working and they can be saved. 

Finally, use `iptables-save` to save the rules. If `iptables-save` is not installed, most package managers have it available. For example, in Ubuntu use: 
```
sudo apt update

sudo apt install iptables-save
```

Once installed, just run the `iptables-save` command with `sudo`. 
```
sudo iptables-save
```

## Reflection
[Short reflection on project - Remove this afterwards]