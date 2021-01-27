# Building a custom app hosted on Cisco IOX-XE platforms.  
This app will check for a config change and post the diff to a Webex Teams Rooms.  The app will also run on a Cisco IOS-XE switch or router, so no additional server resources are needed.

## Setup Overview
This app uses the following:
- Webex Teams
  - A Cisco Communications and Messaging Application.
- EEM in IOS-XE
  - The Embedded Event manager (EEM) is a software component of cisco IOS-XE that can track and classify events that take place and can help you to automate tasks.
- GuestShell in IOS-XE
  - The ability to execute Python code directly on a Cisco Switch is a part of the Application Hosting capabilities provided by GuestShell.  GuestShell is a containerized Linux runtime environment in which you can install and run applications, such as Python scripts.  From within GuestShell, you have access to the networks of the host platform, bootflash and IOS CLI.  GuestShell is isolated from the underlying host software to prevent interference of the core network functions of the device.
- Python Script 
  - This is the part of the app that will process and post the config diff to Webex Teams.


## Webex Incoming Webhooks Integration on Webex App Hub - Preferred method in this usecase
- Webex Teams
  - Create a Webex Teams room where the config diffs will get posted.
  - Connect the *Incoming Webhooks* App in the Webex Teams client.
  - Name and select the Webex Teams room you created.
  - Copy the webhook URL to the python module, 'mytokens.py'.
- Microsoft Team
  - Create a team where the config diffs will be posted.
  - Add the *Incoming Webhook* App in the Microsoft Teams client.
  - Name and add the app to the team you created.
  - Copy the webhook URL to the python module, 'mytokens.py'.
- Slack
  - Create a Channel where the config diffs will be posted.
  - Add the *Incoming Webhooks* App in the Slack client.
  - Choose a custom name and select the channel you created.
  - Copy the Webhook URL to the python module, 'mytokens.py'.

## Webex Bot Setup - Optional alternative to using incoming webhooks
- Create a webex Teams room and get the Room Id.
  - This is where the bot will be posting messages about the config diff on the switch.
  - Copy room id to the python module, 'mytokens.py'.
  - See references below for information on getting the Room Id.
- Create a simple bot and write down the access token.
  - Version 1 of this bot is not interactive.  The bot is only used to post messages to the WebEx Teams Room.
  - Copy the bot's Access Token to the python module, 'mytokens.py'.
  - See references below for information on creating a bot.

## EEM Setup
This is the IOS-XE Configuration for EEM Applet.
  ```
  csr1000v# conf t
  csr1000v(config)# event manager applet test
  csr1000v(config-applet)# event syslog pattern "%SYS-5-CONFIG_I: Configured from" maxrun 200
  csr1000v(config-applet)# action 0.0 cli command "en"
  csr1000v(config-applet)# action 1.0 cli command "guestshell run python3 configDiff.py"
  csr1000v(config-applet)# end
  ```

## GuestShell Setup
- IOX needs to be enable on the IOX-XE platform for GuestShell.
  ```
  csr1000v# conf t
  Enter configuration commands, one per line.  End with CNTL/Z.
  csr1000v(config)# iox
  csr1000v(config)# end
  ```
  
- A VirtualPortGroup is used to enable the communication between IOS XE and the GuestShell container. 
  ```
  csr1000v# conf t
  csr1000v(config)# interface VirtualPortGroup 0
  csr1000v(config-if)# ip address 192.168.1.1 255.255.255.0
  csr1000v(config-if)# end
  ```

- Configure the network settings that will get passed to GuestShell when it's enabled.  
  ```
  csr1000v# conf t
  csr1000v(config)# app-hosting appid guestshell
  csr1000v(config-app-hosting)# vnic gateway1 virtualportgroup 0 guest-interface 0 guest-ipaddress 192.168.1.2 netmask 255.255.255.0 gateway 192.168.1.1 name-server 208.67.222.222
  csr1000v(config-app-hosting)# end
  ```

- Configure NAT if an access from the container to the outside world is needed.
  ```
  csr1000v# conf t
  csr1000v(config)# interface VirtualPortGroup0
  csr1000v(config-if)#  ip nat inside
  !
  csr1000v(config-if)# interface GigabitEthernet1
  csr1000v(config-if)#  ip nat outside
  csr1000v(config-if)# exit
  !
  csr1000v(config)# ip access-list extended NAT-ACL
  csr1000v(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any
  !
  csr1000v(config-if)# exit
  csr1000v(config)# ip nat inside source list NAT-ACL interface GigabitEthernet1 overload
  csr1000v(config)# end
  ```

- Enable GuestShell on the IOX-XE platform.
  ```
  csr1000v# guestshell enable
  ```

- Enter GuestShell to install python and some needed modules.
  ```
  csr1000v# guestshell
  ```

- Optional: We have already defined a DNS Name Server in the app-hosting config for GuestShell, so this step isn't needed. But if you didn't want to configure DNS from IOX-XE, you could configure it directly in the GuestShell environment.  
  ```
  [guestshell@guestshell ~]$ echo "nameserver 208.67.222.222" | sudo tee --append /etc/resolv.conf
  ```
  
- Depending on the IOS-XE platform and version, you may need to install python and some additional utilities.
  ```
  [guestshell@guestshell ~]$ sudo yum update -y
  [guestshell@guestshell ~]$ sudo yum install -y nano python3 epel-release
  [guestshell@guestshell ~]$ sudo yum search pip | grep python3
  [guestshell@guestshell ~]$ sudo yum install -y python3-pip
  ```

- Install the python requests module, and use the optional proxy, if needed.
  ```
  [guestshell@guestshell ~]$ sudo pip3 install --proxy proxy.server.com:8080 requests
  ```
  
- Copy the python script to the EEM user policy directory.  
  - You can copy the script to a directory in GuestShell or you can create a directory on the flash from the IOS-XE CLI.
  - In the EEM config above, the script is located in the home path on GuestShell.
  - If you would like to copy the script to the bootflash, use the absolute path in the EEM config.

- Exit GuestShell and return to IOS-XE
  ```
  [guestshell@guestshell ~]$ exit
  ```
 
**NOTE:** The guestshell environment will persist across reboots.  To return to a default state, destory the guestshell and enable guestshell again.


## Optional - System Proxy Settings for GuestShell
If a proxy server is needed in your enviroment, you'll need to configure the following proxy settings in GuestShell. 

- Create a proxy.sh shell script to add the proxy settings to the system profile.
  ```
  [guestshell@guestshell ~]$ sudo nano /etc/profile.d/proxy.sh
  ```
  
- Add the following parameters in to proxy.sh shell script.
  ```  
  PROXY_URL="http://proxy.server.com:8080/"
  export http_proxy="$PROXY_URL"
  export https_proxy="$PROXY_URL"
  export ftp_proxy="$PROXY_URL"
  export no_proxy="127.0.0.1,localhost"
  export HTTP_PROXY="$PROXY_URL"
  export HTTPS_PROXY="$PROXY_URL"
  export FTP_PROXY="$PROXY_URL"
  export NO_PROXY="127.0.0.1,localhost"
  ```
  
- Source the profile to activate the proxy settings.
  ```
  [guestshell@guestshell ~]$ source /etc/profile 
  [guestshell@guestshell ~]$ env | grep -i proxy
  ```

-  Configure the proxy server for the Yum package manager.
  ```
  [guestshell@guestshell ~]$ echo "proxy=proxy.server.com:8080" | sudo tee --append /etc/yum.conf

  ```

## Run the app
Time to run the app by making a configuration change on the switch. Login to WebEx Teams and check your Teams room for the message.

**NOTE:** Be sure to exit configuration mode since EEM is looking for a specific syslog pattern.


## References
- Many thanks to Patrick Mosimann and the Cisco DevNet Team for sharing their scripts that was the basis for our app:  
  ```
  https://github.com/CiscoDevNet/python_code_samples_network/tree/master/eem_configdiff_to_spark
  ```

- Thanks to Ashish (ashirkar) for his blog post on EEM:
  ```
  https://community.cisco.com/t5/networking-documents/cisco-eem-basic-overview-and-sample-configurations/ta-p/3148479#:~:text=The%20EEM(Embedded%20Event%20manager,minor%20enhancements%20and%20create%20workarounds
  ```

- Thanks to Hank Preston for his blog post on GuestShell:
  ```
  https://community.cisco.com/t5/developer-general-blogs/introducing-python-and-guest-shell-on-ios-xe-16-5/ba-p/3661394
  ```

- Incoming Webhooks Integration on Cisco Webex App Hub
  ```
  https://apphub.webex.com/messaging/applications/incoming-webhooks-cisco-systems-38054
  ```

- Cisco Reference link to create a WebEx Teams Bot:
  ```
  https://developer.webex.com/docs/bots
  ```

- Cisco DevNet Learning Lab reference for getting the Webex Room Id: 
  ```
  https://developer.cisco.com/learning/lab/collab-spark-chatops-bot-itp/step/2
  ```
  
- GuestShell Learning Lab on Cisco DevNet is a good resource for learning more about GuestShell and getting more details on how to configure it.
  ```
  https://developer.cisco.com/learning/modules/net_app_hosting

  ```
