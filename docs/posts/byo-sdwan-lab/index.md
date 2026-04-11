---
date: 2025-05-12
authors:
 - billyzoellers
categories:
    - Cisco
description: Use Cisco Modeling Labs to spin up a fully functional Cisco Catalyst SD-WAN lab enviornment. Perfect for testing policies before deploying to production OR studying for Cisco certifications.
slug: byo-sdwan-lab
draft: true
---

# Build your own Catalyst SD-WAN lab

## Prereq: Cisco Modeling Labs
This guide assumes that you have a running, licensed instance of [Cisco Modeling Labs](https://developer.cisco.com/docs/modeling-labs/introduction/) (CML) avilable, running version 2.7 or greater.

CML is available with multiple license types, including a free version. You'll need a *Personal* or *Enterprise* license to simulated a Catalyst SD-WAN enviornment.

??? question "I need help getting started with CML"
    This guide assumes you already have CML up and running. If not, that's OK! Check out these resources to get started:

    - [Cisco U - Introduction to Network Simulations](https://u.cisco.com/paths/introduction-network-simulations-with-cisco-modeling-labs-243) - This **FREE** Cisco U course will help you get started with CML.
    - [Introduction Docs](https://developer.cisco.com/docs/modeling-labs/introduction/#introduction-to-cisco-modeling-labs) - The CML documentation is publically available and very detailed. This is a great place to start.

??? question "Why can't I use the free license?"
    The CML Free license does not include IOS-XE, which you will need for simulated Catalyst SD-WAN WAN Edges. It is also limited to 5 simulated nodes.

??? question "Where can I get a CML license?"
    *Personal* and *Personal Plus* licenses can be purchased from the Cisco Learning Network store. A [Personal](https://learningnetworkstore.cisco.com/cisco-modeling-labs-personal/cisco-modeling-labs-personal/CML-PERSONAL.html) license allows for 20 simulated nodes, while a [Personal Plus](https://learningnetworkstore.cisco.com/cisco-modeling-labs-personal/cisco-modeling-labs-personal-plus/CML-PERSONAL-PLUS.html) allows for 40.

    You can also utilize Cisco Learning Credits to purchase a Personal license. If multiple users will need access to CML, you can work with your employer and Cisco Account Team to aquire an Enterprise license.

## Prereq: Python
The SD-WAN Lab Deployment Tool is built using Python. Python 3.9.2 or newer will be required on a macOS or Linux system to run this tool. Windows users can use Windows Sybsystem for Linux.

!!! note "Note"
    The Catalyst SD-WAN Deployment Tool is only used to initially provision a simulated Catalyst SD-WAN enviornment. It should be deployed on a workstation with connectivity to the CML enviornment. It does **not** need to be run on the CML server, or on a server that will remain powered on.

## Getting Started

Python best practices suggest creating a [virtual enviornment](https://docs.python.org/3/library/venv.html) to contain an application and it's associated packages.

### Create a new virtual enviornment
Execute these commands in a terminal window to create new Python virtual enviornment.
``` bash
mkdir sdwan-lab # (1)
cd sdwan-lab # (2)
python3 -m venv venv # (3)
source venv/bin/activate # (4)
```

1. Use *mkdir* to create a folder named 'sdwan-lab'
2. Use *cd* change the current working directory to 'sdwan-lab'
3. Create a Python virtual enviornment (or *venv*) named 'venv'
4. Activate the virtual enviornment named 'venv'

The command prompt should now look something like this:
``` bash
(venv) bzoeller@BZOELLER-MAC sdwan-lab %
```

!!! success
    The prefix `(venv)` in the terminal means that future commands will be executed within the Python virtual enviornment.

### Install the Catalyst SD-WAN Deployment Tool
Next, use these commands to update the virtual enviornment and install the Catalyst SD-WAN Deployment Tool:
```bash
pip install --upgrade pip setuptools
pip install --upgrade catalyst-sdwan-lab
```

The Catalyst SD-WAN Deployment Tool should now be ready for use. Test it out with this command:
```bash
sdwan-lab --version
```

!!! success
    If the output returns `SD-WAN Lab, version 2.0.15`, the tool is ready for use!

    Example:
    ```bash
    (venv) bzoeller@BZOELLER-MAC sdwan-lab % sdwan-lab --version
    SD-WAN Lab, version 2.0.15
    ```

## Deploy an SD-WAN Fabric

Now that the Catalyst SD-WAN Deployment Tool has been installed, it can be used to deploy one or more simulated SD-WAN fabrics.

```bash

```


```bash
sdwan-lab \
    --cml cml.bzoel.io \
    --user admin \
    --password cmlPassword \
    --verbose \
    deploy 20.12.4 \
    --lab SDWAN-20-12-4 \
    --manager 172.31.252.225 \
    --mmask 255.255.255.0 \
    --mgateway 172.31.252.1 \
    --muser admin \
    --mpassword vManagePassword

```

## References

