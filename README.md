# Splunk & AD Lab Environment

## Table of Contents
- [Splunk & AD Lab Network Diagram](README.md#splunk--ad-lab-network-diagram)
- [Description](README.md#description)
- [Objective](README.md#objective)
- [Skills Learned](README.md#skills-learned)
- [Tools Used](README.md#tools-used)
- [Part 1 - VM Installation](README.md#part-1--vm-installation)
- [Part 2 - Network Configuration](README.md#part-2--network-configration)
- [Part 3 - Creating Reports and Alerts using Splunk](README.md#part-3---creating-reports-and-alerts-using-splunk)
- [Conclusion](README.md#conclusion)
  
## Splunk & AD Lab Network Diagram
![SOC Lab Environment](images/SOC%20Env.png)

## Description
I built a SOC Monitoring Lab, a small virtual SOC on VirtualBox. I deployed Ubuntu Server (Splunk), Windows Server with a domain controller, Windows 10 as a target machine, and Kali Linux for attacking purposes.

I configured Splunk and Sysmon, forwarded Windows event logs to the indexer, executed controlled attacks from Kali, and created real-time alerts, and scheduled reports to detect, monitor, and investigate suspicious activity.
<br></br>

## Objective
- The objective of this lab is to provide a hands-on environment for practicing cybersecurity. Using VirtualBox, it runs Windows 10, Kali Linux, Windows Server, and Ubuntu Server to cover skills like network setup, endpoint monitoring, and security tool deployment with Splunk and Sysmon. 

- The lab includes offensive testing with Crowbar and Active Directory integration. On the defensive side, it focuses on installing and configuring Splunk, forwarding logs, building reports, and setting up real-time alerts. 
<br></br>

## Skills Learned

#### Active Directory & Windows Security
- Configure and manage AD domains, organizational units (OUs), and user accounts.
#### Splunk SIEM
- Install, configure, and use Splunk for log collection, indexing, and searching.
#### Network Administration
- Configure IP addresses and troubleshoot connectivity issues (ping, DNS, RDP).
#### Log Analysis
- Interpret Windows and network logs to detect suspicious activity and incidents.
#### Problem-Solving
- Develop hands-on troubleshooting and investigative skills for security monitoring.
<br></br>

## Tools Used

#### Windows Server & Windows 10
- Active Directory Domain Controller and client machine.
#### Ubuntu Server
- Hosting Splunk for log collection and analysis.
#### Splunk
- SIEM platform for log ingestion and searching.
#### Kali Linux
- Attacker machine for simulating attacks.
#### Sysmon
- Collecting and analyzing Windows security logs.
#### crowbar
- A brute-force attack tool
<br></br>

## Part 1 – VM Installation

#### 1. Install Oracle VM VirtualBox
- Go to [virtualbox.org](https://www.virtualbox.org/)
- Download the latest version for your OS and complete the installation.

#### 2. Install Windows Server 2022
- Download the ISO from https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022
- In VirtualBox, create a new VM based on your computer resources.

#### 3. Install Ubuntu Server (Splunk Host)
- Download Ubuntu Server 22.04 LTS from https://ubuntu.com/server
- In VirtualBox, create a new VM based on your computer resources.

#### 4. Install Windows 10
- Download the Windows 10 ISO https://www.microsoft.com/en-ca/software-download/windows10
- In VirtualBox, create a new VM based on your computer resources.

#### 5. Install Kali Linux
- Download the VirtualBox image from [kali.org](https://www.kali.org/get-kali/#kali-virtual-machines)
- In VirtualBox, create a new VM based on your computer resources.
<br></br>

## Part 2 — Network Configration

#### 2.1 — Create NAT network and assign to VMs
- In VirtualBox: Tools → Network → NAT Networks → Create. Example IPv4 prefix: 192.168.10.0/24. Give the network a name and Apply.
- For each VM: Settings → Network → Attached to: NAT Network
- select SOC-NAT. Install Windows only (advanced), and complete setup.

#### 2.2 — Set static IP on Splunk (Ubuntu)
Open a terminal on the Ubuntu VM and edit netplan:
`sudo nano /etc/netplan/00-installer-config.yaml` replace or update the file to:
```
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.10.10/24]
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.10.1
  version: 2
```
Apply the configuration and verify `sudo netplan apply`

#### 2.3 — Install Splunk on Ubuntu
- Download Splunk Enterprise .deb from splunk.com (transfer via shared folder or SCP).
- Install VirtualBox guest tools for shared folders `sudo apt-get install virtualbox-guest-utils`
- Mount shared folder (if used)
```
sudo mkdir /mnt/share
sudo mount -t vboxsf -o uid=1000,gid=1000 <shared-folder-name> /mnt/share
ls -la /mnt/share
```
- Install Splunk:
```
sudo dpkg -i /mnt/share/splunk-<version>.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```
- In Splunk Web: Settings → Indexes → New Index → name: endpoint.
- Settings → Forwarding & receiving → Configure receiving → New Receiving Port → set port 9997.

#### 2.3 Configure Windows Server
- Rename the server: ADDC01 → Restart.
- Configure the domain `splunklab.local`:
    - Open Server Manager → Add Roles and Features.
    - Install Active Directory Domain Services (AD DS).
    - Promote server to Domain Controller → Select Add a new forest → Enter domain: `splunklab.local`.
    - Restart and log in as SPLUNKLAB\Administrator.
    - Verify in Active Directory Users and Computers (ADUC) that the domain `splunklab.local` exists.
    - Make OU name IT and create users, for example (jsmith, tsmith) 
- Set static IPv4 on the server’s adapter:
  - IP: 192.168.10.7
  - Subnet mask: 255.255.255.0
  - Default gateway: 192.168.10.1
  - Preferred DNS: 8.8.8.8
- Install Splunk Universal Forwarder on the server and point to 192.168.10.10:9997.
- Install the sysmon app from Splunk apps installation.

#### 2.4 Configure Windows 10 

- Rename PC (optional): Settings → About → Rename this PC → TARGET-PC → Restart.
- Join the Domain (splunklab.local):
  - Right-click This PC → Properties.
  - Click Change settings (under Computer name, domain, and workgroup).
  - Click Change → Select Domain.
  - Enter: `splunklab.local`.
  - Enter Domain Administrator credentials (SPLUNKLAB\Administrator).
  - On success → “Welcome to the `splunklab.local` domain” message appears.
  - Restart PC.
  - Log in using domain credentials (SPLUNKLAB\Administrator or any created domain user).
- Set static IPv4: Network icon → Open Network & Internet Settings → Change adapter options → Right-click adapter → Properties → IPv4 → Use the following IP address
  - IP: 192.168.10.100
  - Subnet mask: 255.255.255.0
  - Default gateway: 192.168.10.1
  - Preferred DNS: 8.8.8.8
- Verify `ipconfig shows 192.168.10.100`
- From a Windows browser, visit `http://192.168.10.10:8000` to confirm Splunk is reachable.
- Verify logs in Splunk: index=endpoint — you should see TARGET-PC and ADDC01 as hosts.

#### 2.5 Install Splunk Universal Forwarder and Sysmon
- Download Splunk Forwarder from http://splunk.com.
- Set up Splunk Forwarder and in the (Receiving Indexer) set `192.168.10.10:9997`
- Configure inputs on the forwarder: create `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf` with: 
```
[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```
- Download Sysmon (Sysinternals) and a recommended config sysmonconfig.xml from sysmon-modular.
- In PowerShell (as Administrator), install Sysmon: `.\Sysmon64.exe -i .\sysmonconfig.xml`
- In Splunk Web, search index=endpoint to confirm logs arrive.
- Do these steps on Active Directory and the Windows 10 machines.

 #### 2.6 Kali Linux
 Set Static IP
- Power on the Kali Linux VM and log in.
- Right-click the network/connection icon (top-right).
- Select Edit Connections → Wired connection 1 → Edit selected connection.
- Go to IPv4 Settings:
  - Method: Manual
  - Click Add and enter:
  - Address: 192.168.10.250 - Netmask: 255.255.255.0 (or /24) - Gateway: 192.168.10.1 - DNS servers: 8.8.8.8
  - Click Save.
 
Update Kali
- Run system update & upgrade: `sudo apt-get update && sudo apt-get upgrade -y`
  
Perform a brute-force test using `crowbar` over any user that you created.
- `sudo crowbar -b rdp -u tsmith -C passwords.txt -s 192.168.10.100/32 `

## Part 3 —  Creating Reports and Alerts using Splunk
#### 3.1 Creating Reports 
1. Go to Search & Reporting.
2. Run your search. `index=endpoint EventCode=4625`
3. Click Save As → Report.
4. Name the report.
5. Choose Scheduled or Real-time.
6. Save and test the alert.

![SOC Lab Environment](images/report_image/creating_report.png)
![SOC Lab Environment](images/report_image/creating_report2.png)
![SOC Lab Environment](images/report_image/creating_report3.png)

To view/manage reports: App → Search&Reports → Reports

#### 3.2 Creating Alerts
1. Go to Search & Reporting.
2. Run your search. `index=endpoint EventCode=4625`
3. Click Save As → Alert.
4. Name the alert.
5. Choose Scheduled or Real-time.
6. Set Trigger conditions.
7. Add Alert actions.
8. Save and test the alert.

![SOC Lab Environment](images/alert_image/creating_alert1.png)
![SOC Lab Environment](images/alert_image/creating_alert2.png)
![SOC Lab Environment](images/alert_image//alert_info.png)

To view/manage reports: App → Search&Reports → Alerts

#### 3.3 Check Alerts and Report Output
- You will find the path of alerts and reports in: Settings → Lookup  → Lookup table files 
- In the Splunk Server, go to this path and check the alerts and reports
  
![SOC Lab Environment](images/report_alerts_path.png)

#### 3.4 Check Alerts and Report Output
Checking the path of the report and alerts to see the output 

![SOC Lab Environment](images/report.png)
<br></br>

## Conclusion
This project demonstrates my ability to design and implement a complete SOC monitoring lab using Splunk and Active Directory.  
It highlights practical skills in search creation and alert configuration.  

⭐ Feel free to explore, share feedback, or suggest improvements - always learning and evolving! 🙌





