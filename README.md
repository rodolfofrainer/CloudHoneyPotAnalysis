# SOC Microsoft Sentinel Analyst Project

This project aims to practice the creation, configuration, deployment and analysis of a virtual machine exposed to the internet.
The pre requisit for this project is to have an Azure account.

- [SOC Microsoft Sentinel Analyst Project](#soc-microsoft-sentinel-analyst-project)
  - [Resouce Group](#resouce-group)
    - [Creation](#creation)
  - [Virtual Network](#virtual-network)
    - [Creation](#creation-1)
  - [Virtual Machine](#virtual-machine)
    - [Creation](#creation-2)
  - [Setting Firewall Rules](#setting-firewall-rules)
    - [Adding a inbound rule](#adding-a-inbound-rule)
  - [Logging into the VM](#logging-into-the-vm)
    - [Disable Windows Firewall](#disable-windows-firewall)
  - [Pinging honeypot](#pinging-honeypot)
  - [Checking the logs](#checking-the-logs)
    - [After logging in successfully](#after-logging-in-successfully)
  - [Analytic workspace](#analytic-workspace)
    - [Sentinel](#sentinel)
  - [Back to Sentinel](#back-to-sentinel)
    - [Importing Data to SIEM](#importing-data-to-siem)
    - [Back to the Logs visualization](#back-to-the-logs-visualization)
    - [Creating a map](#creating-a-map)
  - [Special Thanks](#special-thanks)

## Resouce Group

Resource groups are containers that hold related resources, providing a way to manage and organize resources logically.

### Creation

1. Search for the term "Resource Group" in the search bar
2. Click the "Create button"
3. Select a subscription, Group name and a Region (The one closest to your physical location)
4. Click "Review + create"; this will run and make sure that all of the configuration is in order
5. Click "Create"

With this the Resource group is created. In the case of this project it will be called "RG-SOC-Lab"

## Virtual Network

Virtual networks are a virtualized concept of how the network solution traditionally works. Through software Azure can send the correct data packets to virtual machines.

### Creation

1. Using the search bar look for "Virtual networks"
2. Click the "Create button"
3. Select a Subscription and a Resouce group, in this step we are using the last created resource group, "RG-SOC-Lab"; you will also have to select the region (the same as the resource group is best) and give the network a name in our case that will be "Vnet-SOC-Lab"
4. No other configuration will be changed for thi project so you can create the resource clicking on "Review" button and after the check is done, click the "Create" button after

## Virtual Machine

The virtual machine, a windows 10 in our case, will be used as the honeypot, it will be configured with a "juicy" name enticing attackers to try and break into the machine. With that we will forward the logs into a different resource in order to query and study them.

### Creation

1. "Virtual Machines" in the search bar
2. Create button
3. Fill the information requested. Our machine will be called "Corp-Net-WestGem-003" and the image is "Windows 10 Pro". **Attention:** When picking the "size", some of the sizes will consume a lot of resources and you might end up paying more the the estimation. Be sure to turn the VM off when not in use anymore.
4. For username I will be using "labuser" and a **SECURE** password
5. Almost every other configuration will stay the same with the exception of "Delete public IP and NIC when VM is deleted", which will be checked, and disable boot diagnostics.
6. Create the VM

## Setting Firewall Rules

A Firewall, in this case, is a software-based solution that will allow or not allow data or communication across the network/networks to take place.

### Adding a inbound rule

1. Select the RG-SOC-Lab (Resource groups tab) and from the resources shown pick the option with type "network security group".
2. On the left bar -> Settings -> Inbound security rules -> Add

   ![InboundRuleCreation](/images/inboundRuleCreation.jpg)

_The name has a prefix of "DANGER" because all destination ports are open, that is incredible dangerous and should to be undone as soon as not needed anymore_

## Logging into the VM

To log into the VM a program or an agent. In Windows the remote desktop can be used. On linux I am using the "Remote Desktop Manager".
Provide the user and password from when the VM was created and the will allow you to log in.

### Disable Windows Firewall

1. Ater logging into the VM, you might need to go through a bit of the windows setup, then on the search bar search for the term "wf.msc".
2. On the overview click "Windows Defender Firewall Properties"
3. Turn the "Firewall state" off on all tabs.

## Pinging honeypot

Now the machine should be completely exposed to the internet

1. Navigate to Azure's portal
2. Select your VM and look for the overview of the resource, the public IP Adddress should be visible there

![IpLocation](/images/IpLocation.jpg)

3. Using the terminal or cmd on your own machine use the command `ping <VM IP Address>`; it should be able to reach the destination.

_Now the machine is exposed, it might take a few minutes but eventually someone WILL ping it as well and try to connect to it using all sort of exploits_

## Checking the logs

At this stage I would recommend trying to log with wrong credentials to ensure that not anyone is logging into the machine without proper credentials.

### After logging in successfully

1. In the VMs search bar look for "Event Viewer"

   _The event viewer is essentially the log visualitaion tool that windows use_

2. On the left bar Windows Logs -> Security

   _A list of logs will be displayed, they show successfull and unsuccessfull logins_

3. Filter the logs, and look for the event ID 4625, that is the ID for failed login. If there's traffic not generated from you already that means that attackers have picked on your IP.

   _It is not unusual that a large amount of traffic comes through very quickly._

4. Double clicking the event will display a range of information, including "Account Name", "Source Network Address", "Logged" (timestamp), etc.

![LogExample](/images/example%20log.jpg)

## Analytic workspace

1. On Azure's portal navigate to "Log analytics workspace" and create a new workspace

   ### Sentinel

   1. Once the analitics workspace is ready navigate to microsoft sentinel (this feature has a 31-day free trial) and create an instance.
   2. On the left bar > Content Management > Content Hub > Search bar > select "microsoft security events" > click "Install"
   3. Refresh the page, find "microsoft security events" on the list and click "Manage"
   4. Find and click "Windows security Events via AMA" > "Open connector page" button > Create data collection rule
   5. Fill in basic information > Next > On Resources tabs select your VM (It might be collapsed on the menu) > Select "All security Events" > Create the Rule

2. On the left bar of the VM click logs

   _A new window will render_

3. Change to KQL mode

![KqlMode](/images/kqlMode.jpg)

4. We can now query the data using KQL, `SecurityEvent |where EventID == 4625`
   At this moment this is returning close to 1000 results

5. Filtering the information being displayed with `|project` allows for a more easily interpretation of the data. The command I using is `SecurityEvent | where EventID == 4625 |project TimeGenerated,Account,TargetAccount, IpAddress, IpPort` which provides me with this results

![beingProbed](/images/beingProbed.jpg)

6. Looking at the image we can deduct that IpAddress `94.41.109.172` is checking my ports trying to log in as the administrator. The ports started a 51930 and probing if there is any exploit he can take advantage of.

7. Using [geolookup](https://www.iplocation.net/ip-lookup) I can see some additional information from the attacker.

![attackerInfo](/images/attackerInfo.jpg)

## Back to Sentinel

Analysing individual IPs is not practical, luckily Sentinel allow us locate individuals, in the projects files there's a csv called "geoip-summarized", it will be important in a second

### Importing Data to SIEM

On Azure's Sentinel select the SIEM, then on the left bar > configuration > watchlist > New > _fill the information, Im using "geoip" for name and alias_> Next > Upload cvs file and select the SearchKey as "network" > Create

_This **WILL** take a while_

### Back to the Logs visualization

`_GetWatchlist("geoip")` allows us to query the information we just uploaded

![geopIp](/images/geoipVis.jpg)

1. The folloing command will be used `let GeoIPDB_FULL = _GetWatchlist("geoip"); let WindowsEvents = SecurityEvent |where IpAddress == <attacker IP address>| where EventID == 4625 | order by TimeGenerated desc | evaluate ipv4_lookup(GeoIPDB_FULL, IpAdress, network); WindosEvents`

2. That will provide us with a list containing city names and country names in this case:

![parsedInfo](/images/parsedInformation.jpg)

The IP Address range is apparently from Portugal, as one IPS in Portugal has a range of IPs that contains "94.41.109.172"

_\* We can simplify the output with this command_
`let GeoIPDB_FULL = _GetWatchlist("geoip"); let WindowsEvents = SecurityEvent | where IpAddress == "94.41.109.172"| where EventID == 4625 | order by TimeGenerated desc | evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network); WindowsEvents |project TimeGenerated, Computer, AttackerIp = IpAddress, cityname, countryname, latitude, longitude`

### Creating a map

To create a map a Sentinel workbook has to be created

1. Add a workbook > Edit > Delete all pre-populated graphs
2. Click "Add" > Add query > Advanced Editor > Replace data with content from map.json > Done editing
3. The map can be visualized

![ipMap](/images/ipMap.jpg)

## Special Thanks

- Josh Madakor has developed this fantastic tutorial to help people to gain experience in the field, free
