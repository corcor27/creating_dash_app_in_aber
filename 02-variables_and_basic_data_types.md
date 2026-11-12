---
title: Foundation & The FAIR Principles
teaching: 75
exercises: 0
---

::::::::::::::::::::::::::::::::::::::: objectives
- "Connect to a remote HPC system."
- "Understand the general HPC system architecture."
 
::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- "What is an HPC system?"
- "How does an HPC system work?"
- "How do I log on to a remote HPC system?"

::::::::::::::::::::::::::::::::::::::::::::::::::

## What is an HPC system?

The words "cloud", "cluster", and "high-performance computing" are used a lot in different contexts
and with varying degrees of correctness. So what do they mean exactly? And more importantly, how do
we use them for our work?

The *cloud* is a generic term commonly used to refer to remote computing resources of any kind --
that is, any computers that you use but are not right in front of you. Cloud can refer to
machines serving websites, providing shared storage, providing webservices (such as e-mail or social
media platforms), as well as more traditional "compute" resources. An *HPC system* on the other hand,
is a term used to describe a network of computers. The computers in a cluster typically share a common
purpose, and are used to accomplish tasks that might otherwise be too big for any one computer.

## Network access

## Network Access Requirements

Over the past year, the university network has been upgraded. As a result:  

- **General systems connected to eduroam** are now restricted to internet access only (e.g., web browsing).  
- **Aber IS-supported systems** can connect to the server automatically.  
- **All other systems** (including CS desktops) require the **Aberystwyth VPN** to access the server.  

**VPN portals:**  
- Wi-Fi (eduroam): `gp-vpn.aber.ac.uk`  
- Ethernet: `gp-vpn-int.aber.ac.uk`  

If you encounter issues setting up the VPN, please contact **Information Services (IS)** by raising a support ticket at: [is@aber.ac.uk](mailto:is@aber.ac.uk).  

## Logging in

Go ahead and log in to the Aberystwyth HPC cluster: {{ site.host_name }} at {{ site.host_location }}.
```bash
{{ site.local_prompt }} ssh yourUsername@{{ site.host_login }}
```


Remember to replace `yourUsername` with the username supplied by the instructors. You will be asked for
your password. But watch out, the characters you type are not displayed on the screen.

:::::::::::::::::::::::::::::::::::::::::  callout

## How do I specify common `ssh` options

To reduce issues with connecting it can be useful to fix the settings to be used.
For example `ssh` will try every key in your `~/.ssh` directory, and can trigger the
banning of your machine from accessing the system if you have more than 1 key.

On a Mac/Linux laptop/desktop you can add settings to `$HOME/.ssh/config` such as:

```bash
Host login01 bert.aber.ac.uk
Hostname bert.aber.ac.uk
User username
IdentityFile ~/.ssh/id_rsa-aber
```
On Windows depending on software these settings can be set in the menu settings.

Alternatively use `PreferredAuthentications password` to force ssh to always use only a password.  This may be needed if ssh keys are available 
on your own computer but not used for Hawk and can trigger the banning of your machine when attempts with the keys are made.

Other option is `SetEnv LANG=C` can be useful if a locale setting on your desktop is not supported on Hawk for some reason and causes text encoding issues.
::::::::::::::::::::::::::::::::::::::::::::::::::

You are logging in using a program known as the secure shell or `ssh`. 
This establishes a temporary encrypted connection between your laptop and `{{ site.host_login }}`.
The word before the `@` symbol, e.g. `yourUsername` here, is the user account name that Lola has access 
permissions for on the cluster. 

:::::::::::::::::::::::::::::::::::::::::  callout
## Where do I get this `ssh` from ?
On Linux and/or macOS, the `ssh` command line utility is almost always pre-installed. Open a
terminal and type `ssh --help` to check if that is the case. 

At the time of writing, the openssh support on Microsoft is still very 
[recent](https://blogs.msdn.microsoft.com/powershell/2017/12/15/using-the-openssh-beta-in-windows-10-fall-creators-update-and-windows-server-1709/). 
Alternatives to this are [putty](http://www.putty.org), 
[bitvise SSH](https://www.bitvise.com/ssh-client-download), 
[mRemoteNG](https://mremoteng.org/) or [MobaXterm](https://mobaxterm.mobatek.net/). 
Download it, install it and open the GUI. The GUI asks for your user name and the destination
address or IP of the computer you want to connect to. Once provided, you will be queried for 
your password just like in the example above.
::::::::::::::::::::::::::::::::::::::::::::::::::

## Where are we?

Very often, many users are tempted to think of a high-performance computing installation as one
giant, magical machine. Sometimes, people will assume that the computer they've logged onto is the
entire computing cluster. So what's really happening? What computer have we logged on to? The name
of the current computer we are logged onto can be checked with the `hostname` command. (You may also
notice that the current hostname is also part of our prompt!)

Aber:

```bash
{{ site.host_prompt}} hostname
```


```output
{{ site.host_name }}
```

## Nodes

Individual computers that compose a cluster are typically called *nodes* (although you will also
hear people call them *servers*, *computers* and *machines*). On a cluster, there are different
types of nodes for different types of tasks. The node where you are right now is called the *head
node*, *login node* or *submit node*. A login node serves as an access point to the cluster. As a
gateway, it is well suited for uploading and downloading files, setting up software, and running
quick tests. **It should never be used for doing actual work**.

The real work on a cluster gets done by the *worker* (or *compute*) *nodes*. Worker nodes come in
many shapes and sizes, but generally are dedicated to long or hard tasks that require a lot of
computational resources.

All interaction with the worker nodes is handled by a specialized piece of software called a
scheduler (the scheduler used in this lesson is called {{ site.workshop_shed_name }}). We'll
learn more about how to use the scheduler to submit jobs next, but for now, it can also tell us
more information about the worker
nodes.

For example, we can view all of the worker nodes with the `{{ site.sched_info }}` command **(Same for both sites)**.

```bash
{{ site.host_prompt}} {{ site.sched_info }}
```

Aber:

```output
{% include /snippets/12/info.snip %}
```



There are also specialized machines used for managing disk storage, user authentication, and other
infrastructure-related tasks. Although we do not typically logon to or interact with these machines
directly, they enable a number of key features like ensuring our user account and files are
available throughout the HPC system.

:::::::::::::::::::::::::::::::::::::::::  callout
## Shared file systems
This is an important point to remember: files saved on one node (computer) are often available everywhere on the cluster!
::::::::::::::::::::::::::::::::::::::::::::::::::

## What's in a node? 

All of a HPC system's nodes have the same components as your own laptop or desktop: *CPUs* (sometimes
also called *processors* or *cores*), *memory* (or *RAM*), and *disk* space. CPUs are a computer's
tool for actually running programs and calculations. Information about a current task is stored in
the computer's memory. Disk refers to all storage that can be accessed like a file system. This is
generally storage that can hold data permanently, i.e. data is still there even if the computer has
been restarted.

{% include figure.html url="" max-width="40%" file="/fig/node_anatomy.png" alt="Node anatomy" caption="" %}


:::::::::::::::::::::::::::::::::::::::  challenge
## Explore Your Computer

Try to find out the number of CPUs and amount of memory available on your personal computer.
::::::::::::::::::::::::::::::::::::::::::::::::::


:::::::::::::::::::::::::::::::::::::::  challenge
## Explore The Login Node (Same for both sites)
Now we'll compare the size of your computer with the size of the login node: To see the number of
processors, run:
```bash
{{ site.host_prompt}} nproc --all
```
How about memory? Try running: 

```bash
{{ site.host_prompt}} free -m
```
::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout

## Getting more information (Same for both sites)

You can get more detailed information on both the processors and memory by using different commands.

For more information on processors use `lscpu`
```bash
{{ site.host_prompt}} lscpu
```
For more information on memory you can look in the `/proc/meminfo` file:
```bash
{{ site.host_prompt}} cat /proc/meminfo
```

::::::::::::::::::::::::::::::::::::::::::::::::::

{% include /snippets/12/explore.snip %}

:::::::::::::::::::::::::::::::::::::::  challenge
## Compare Your Computer, the Login Node and the Worker Node

Compare your laptop's number of processors and memory with the numbers you see on the cluster login node and worker node. Discuss the differences with your neighbor. What implications do you think the differences might have on running your research work on the different systems and nodes?
::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout
## Units and Language

A computer's memory and disk are measured in units called *Bytes* (one Byte is 8 bits). As today's files and memory have grown to be large given historic standards, volumes are  noted using the [SI](https://en.wikipedia.org/wiki/International_System_of_Units) prefixes. 

So 1000 Bytes is a Kilobyte (kB), 1000 Kilobytes is a Megabyte, 1000 Megabytes is a Gigabyte etc. 

History and common language have however mixed this notation with a different meaning. When people say "Kilobyte", they mean 1024 Bytes instead. In that spirit, a Megabyte are 1024 Kilobytes. To address this ambiguity, the [International System of Quantities](https://en.wikipedia.org/wiki/International_System_of_Quantities) standardizes the binary prefixes (with base of 1024) by the prefixes kibi, mibi, gibi, etc. For more details, see [here](https://en.wikipedia.org/wiki/Binary_prefix)
::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout
## Differences Between Nodes
Many HPC clusters have a variety of nodes optimized for particular workloads. Some nodes may have larger amount of memory, or specialized resources such as Graphical Processing Units (GPUs).
::::::::::::::::::::::::::::::::::::::::::::::::::

With all of this in mind, we will now cover how to talk to the cluster's scheduler, and use it to
start running our scripts and programs!

## Details about the server

Below is a detailed image showing the configuration of the server along with all its partitions. 

{% include figure.html max-width="80%" file="/fig/hpc_upgrade.png" 
alt="Schematic of how a computer works" caption="" %}

## HPC Hardware Overview

### Bert (Log-in Node) — TDP 120W
- **CPU:** AMD EPYC 7252 (8-Core, 120W)  
- **Memory:** 48GB  

---

### Ernie (x3) — TDP 200W each | Total 600W
- **CPU:** AMD EPYC 7552 (48-Core, 2.2GHz, 200W)  
- **Memory:** 384GB  

---

### hpc-gn-ampere-01 (AM-01) — TDP 2880W
- **GPU:** 8 × NVIDIA A100 (80GB VRAM each, 300W | 2400W total)  
- **CPU:** 2 × AMD EPYC 7F72 (24-Core, 240W | 480W total)  
- **Memory:** 2TB  

---

### hpc-ci-cascade-lake-01 (CL-01) — TDP 410W
- **CPU:** 2 × Intel Xeon Gold 6248R (3.0GHz, 205W | 410W total)  
- **Memory:** 1.5TB  

---

### hpc-gnw-ampere-02 (AM-02) — TDP 880W
- **GPU:** 2 × NVIDIA RTX A6000 (48GB VRAM each, 300W | 600W total)  
- **CPU:** AMD Ryzen Threadripper PRO 3975WX (280W)  
- **Memory:** 128GB  

---

### hpc-gn-ampere-03 (AM-03) — TDP 2800W
- **GPU:** 8 × NVIDIA A100 (80GB VRAM each, 300W | 2400W total)  
- **CPU:** 2 × AMD EPYC 7702 (64-Core, 200W | 400W total)  
- **Memory:** 2TB  

---

### hpc-gn-ampere-04 (AM-04) — TDP 2800W
- **GPU:** 8 × NVIDIA A100 (80GB VRAM each, 300W | 2400W total)  
- **CPU:** 2 × AMD EPYC 7702 (64-Core, 200W | 400W total)  
- **Memory:** 2TB  

---

### hpc-gn-pascal-01 (PA-01) — TDP 625W
- **GPU:** 2 × NVIDIA Tesla P100 (16GB VRAM each, 250W | 500W total)  
- **CPU:** Intel Xeon Gold 6130 (2.1GHz, 125W)  
- **Memory:** 48GB  
- **Note:** Legacy GPU server  

---

### hpc-gn-hopper-01 (HO-01) — TDP 3520W
- **GPU:** 4 × NVIDIA H100 (94GB VRAM each, 700W | 2800W total, configurable to 400W each)  
- **CPU:** 2 × AMD EPYC Genoa 9654 (96-Core, 2.4GHz, 360W | 720W total)  
- **Memory:** 2TB  

---

### Cascade Lake Cluster (CL-02 to CL-06) — TDP 250W (standard) | 750W (optional)
- **Nodes:** CL-02, CL-03, CL-04, CL-05, CL-06  
- **CPU (each):** 2 × Intel Xeon Gold 6254 (3.1GHz, 125W | 250W total)  
- **Memory (each):** 768GB  

---

### hpc-ca-genoa-01 (GE-01) — TDP 115W
- **CPU:** Intel Xeon E5-2620 v4 (2.1GHz, 115W)  
- **Memory:** 1TB  

---

### hpc-gn-ampere-05 (AM-05) — TDP 455W
- **GPU:** 1 × NVIDIA A100 (40GB VRAM, 300W)  
- **CPU:** AMD EPYC 7452 (155W)  
- **Memory:** 768GB  

:::::::::::::::::::::::::::::::::::::::: keypoints

- "An HPC system is a set of networked machines."
- "HPC systems typically provides login nodes and a set of worker nodes."
- "The resources found on independent (worker) nodes can vary in volume and type (amount of RAM, processor architecture, availability of network mounted file systems, etc.)."
- "Files saved on one node are available on all nodes."
::::::::::::::::::::::::::::::::::::::::::::::::::


