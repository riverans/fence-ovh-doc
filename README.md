OVH fencing agent documentation
===============================

Introduction
------------

This documentation helps using OVH fencing agent. Documentation is aimed at a Proxmox installation but can be easily adapted to other HA systems. You are supposed to have been able to setup your Proxmox installation as a cluster and begun to setup it as an HA cluster. As per having a server in OVH datacentre you need a fencing mechanism but OVH does not deliver a direct fencing mechanism. The alternative, which it's coded into this fence agent, is to reboot your machine into the OVH rescue mode.

The documentation, although it's kind of offtopic will also cover how to create your OVH application keys. The reason is that working with these OVH application keys is very recent and, the other reason, is that you need these keys if you want Fencing to work.

The repository that hosts this fencing agent won't contain fencing agent binaries but some auxiliar files which we might mention and link to.

If you need more information on why you need a fencing agent in your HA installation please check: https://fedorahosted.org/cluster/wiki/FAQ/Fencing#fence_manual3

Index
-----

* [Requirements](#requirements)
* [OVH application keys generation](#ovh-application-keys-generation)
* [Manual installation](#manual-installation)
* [Setting up your cluster.conf](#setting-up-your-clusterconf)

Requirements
------------

* **An High Availability (HA) system**. This howto is based mainly in Proxmox. Actually the HA system needs the OVH fencing.
* **OVH dedicated server**. This howto only makes sense if you are using dedicated servers in OVH datacentres.
* **Proxmox setup as a cluster**. We assume that Proxmox is already setup as a [cluster](https://pve.proxmox.com/wiki/Proxmox_VE_2.0_Cluster).
* **Proxmox mostly setup as an HA cluster**. We assume that you have begun to setup Proxmox as an [High Availability cluster](https://pve.proxmox.com/wiki/High_Availability_Cluster) and you are stuck at setting up a fencing agent inside OVH infraestructure.
* **Vrack** . This is actually not strictly needed. However we recommend you to purchase [Vrack](https://www.ovh.co.uk/solutions/vrack/)  to ease Proxmox cluster communication setup. You will also have one dedicated network device for service and another one for communications as most suggested network setups. These are other technical pages about Vrack: [Vrack 1.0](http://help.ovh.co.uk/vrack) , [Vrack 1.5](http://help.ovh.co.uk/VrackInfrastructure) . Vrack 2.0 (Being able to connect servers from different datacentres around the world) is supposed to come soon.
* **RipeVrack** . This is actually not strictly needed. If you are used to setup your virtual machines network configuration as [Bridgeclient in OVH](http://help.ovh.co.uk/BridgeClient) we recommend you to switch to [RipeVrack](http://help.ovh.co.uk/RipeVrack) which it's the most suited for HA (no need to change .254 gateway).
* **OVH application keys**. This is needed in order your OVH fencing agent to work. Don't worry if you don't have these [OVH application keys](https://api.ovh.com/g934.first_step_with_api) because we will cover their generation in this howto.

OVH application keys generation
-------------------------------

### Introduction

OVH introduced a new RESTful based API in 2014. This API has a different approach for authentication than the older SOAPI one. Basically it helps the OVH user to avoid handling their OVH account user and password to their sysadmins.

Why? Because application or software that uses OVH RESTful API use a separated pair of keys for identifying themselves and a third key in addition for being able to interact with the RESTful API.

The other improvement is that the application is not granted all the OVH RESTful API methods to be able to interact with. You can specify which API requests the application can only interact with.

### Requirements

* curl program
  * Each one of the OVH names for each one of the dedicated servers that conform your Proxmox cluster. Eg.
    * `ns1100101.ip-101-01-01.eu`
    * `ns2200202.ip-202-02-02.eu`
    * `ns3300303.ip-003-03-03.eu`

### Who is in charge on generating OVH application keys

The Proxmox HA system sysadmin does not need to deal with OVH application keys. In order to make HA work it only needs **AK**, **AS** and **CK** for the fencing application. Your general sysadmin (the one with right for knowing OVH account user and password) should deal with this **AK**, **AS** and **CK** generation.

Alternatively you can handle OVH account user and password to your Proxmox HA system sysadmin temporarily while it makes: **AK**, **AS** and **CK** and then change the OVH account password.

### Creation of your application keys

Click on the following link: https://eu.api.ovh.com/createApp/, enter your customer ID, your password, and the name of your application. If your bussiness is called **mybu** we recommend you to use: **mybu-fencing-agent** as the application name.

You get two keys:
* the application key, named **AK**, e.g:
`7kbG7Bk7S9Nt7ZSV`

* your secret application key, named **AS**, e.g:
`EXEgWIz07P0HYwtQDs7cNIqCiQaWSuHF`

### Requesting an authentication token from OVH

We will request a token from OVH to enable the OVH fencing agent to make requests to the API. Specifically we will only ask for being able to perform these requests:

* PUT /dedicated/server/{serviceName} (Change boot device)
* POST /dedicated/server/{serviceName}/reboot (Hard reboot your dedicated server)
* GET /dedicated/server/{serviceName}/task/{taskId} (Fetch task details)
* GET /dedicated/server/{serviceName}/boot/ (Fetch server compatible netboots)

In order to perform this request we will use our a home-made script named [fencing-ovh-request-authentication-code.sh](../master/fencing-ovh-request-authentication-code.sh) [(I recommend you to download it as raw file)] (https://github.com/adriangibanelbtactic/fence-ovh-doc/raw/master/fencing-ovh-request-authentication-code.sh).

You will have to edit the file so that it meets your needs.
Please modify:
```
# Variables you should adapt to your own case:
OVH_APPLICATION_KEY="7kbG7Bk7S9Nt7ZSV"
DEDICATED_SERVER_OVH_NAMES="ns1100101.ip-101-01-01.eu ns2200202.ip-202-02-02.eu ns3300303.ip-003-03-03.eu" # Separated by spaces
```
so that you use your own application key (AK) and so that put your dedicated server ovh names separated by spaces.

First we give it execution permissions:
```
chmod +x fencing-ovh-request-authentication-code.sh
```

Then you just need to run it like this:

```
./fencing-ovh-request-authentication-code.sh
```

The response should be similar to:
```
{"validationUrl":
"https://api.ovh.com/auth/?credentialToken=
iQ1joJE0OmSPlUAoSw1IvAPWDeaD87ZM64HEDvYq77IKIxr4bIu6fU8OtrPQEeRh"
,"consumerKey":
"MtSwSrPpNjqfVSmJhLbPyr2i45lSwPU1",
"state":"pendingValidation"}
```
First you need to write your consumer key (CK):
```
MtSwSrPpNjqfVSmJhLbPyr2i45lSwPU1
```

Now you need to visit:

https://api.ovh.com/auth/?credentialToken=iQ1joJE0OmSPlUAoSw1IvAPWDeaD87ZM64HEDvYq77IKIxr4bIu6fU8OtrPQEeRh

to aprove this request.

First of all check the access granted to see if it corresponds to your servers and the forementioned needed requests.

Then enter your customer ID, your password and the validity which we recommend to be **Unlimited** and click in **Authorize access** button.

### Application keys have been generated ###

After seeing that the OVH has validaded the consumer key now you have three important settings to handle to your Proxmox HA system sysadmin:

* Application Key (**AK**): `7kbG7Bk7S9Nt7ZSV`
* Secret Application Key (**AS**): `EXEgWIz07P0HYwtQDs7cNIqCiQaWSuHF`
* Consumer key (**CK**): `MtSwSrPpNjqfVSmJhLbPyr2i45lSwPU1`

Manual installation
-------------------

You only need a manual installation if fence_ovh is not coming by default in Proxmox, on your HA system, or if you feel that its version is outdated.

Note: This manual installation will overwrite `/usr/sbin/fence_ovh` file which might be your distribution default fence script for OVH. Feel free to backup it prior to the manual installation if you want to return back to previous status.

Please use the script: [install_fence_ovh.sh](../master/install_fence_ovh.sh) [(I recommend you to download it as raw file)] (https://github.com/adriangibanelbtactic/fence-ovh-doc/raw/master/install_fence_ovh.sh)

First we give it execution permissions:
```
chmod +x install_fence_ovh.sh
```

Then you just need to run it **as the root user** like this:

```
./install_fence_ovh.sh
```



Setting up your cluster.conf
----------------------------

You can find an example of a cluster.conf based on three big server (Check bigserver explanation later) nodes depicted above here: [cluster_3_nodes.conf](../master/cluster_3_nodes.conf) [(I recommend you to download it as raw file)] (https://github.com/adriangibanelbtactic/fence-ovh-doc/raw/master/cluster_3_nodes.conf).

We will explain each one of the fence_ovh specific parametres

* **port** : **(Required)** Dedicated server OVH name. In the past they were in the form `ns1100101.ovh.net` now they are something similar to: `ns1100101.ip-101-01-01.eu`
* **power_wait**: **(Optional)** Default timeout after asking OVH to reboot is 240 seconds. You can change this timeout with this parametre. Units: seconds. This parametre is ignored when `bigserver` is `on`.
* **login**: **(Required)** Application Key (**AK**): E.g. `7kbG7Bk7S9Nt7ZSV`
* **passwd**: **(Required)** Secret Application Key (**AS**): E.g. `EXEgWIz07P0HYwtQDs7cNIqCiQaWSuHF`
* **ovhcustomerkey**: **(Required)** Consumer key (**CK**): E.g. `MtSwSrPpNjqfVSmJhLbPyr2i45lSwPU1`
* **ovhapilocation**: **(Optional)** Either "EU" for Europe based OVH API or "CA" for Canada based OVH API. Default is "EU".
* **bigserver** : **(Optional)** Either `on` or `off`. When bigserver is on then the script checks for reboot task status 90 seconds after starting it. If the task status is `doing` or `done` it assumes the server is being rebooted ok. This is not as safe as `bigserver = off` which waits 240 seconds (or `power_wait` value) to server to boot into rescue mode and checks it with task status being done. However, as OVH is probably not going to extend reboot task time more than 5 minutes it is the default because most people which will be using the HA fence agent will be using big servers which tend to have an ipmi, a raid card, several network cards which its initialisation make the 5 minutes reboot to timeout. Default is `on`.

As you might guess from cluster.conf the only action you are going to use is "off" which translates into the server booting into rescue mode and checking if it has booted into rescue mode in less than 240 seconds.

Note: If `bigserver` is `off` OVH will fail the reboot the operation if it takes longer than 300 seconds no matter what power_wait value you define here.