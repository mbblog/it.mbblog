---
author: "Mathis Bramkamp"
title: "Using pySROS in a CI/CD Pipeline | Part 1"
date: "2024-07-29"
description: "This blog describes the very first steps of using pySROS in a CI/CD pipeline."
summary: "It is being explained on how pySROS can be used in a CI/CD pipeline. A simple use-case of pulling NE configuration backups is used for this example"
categories: ["Network Automation"]
ShowToc: true
TocOpen: true
---

# Brief introduction into pySROS and CI/CD pipelines

## The pySROS python library

pySROS is a Python module that provides an API for managing and automating Nokia routers running SR OS. It allows network engineers and developers to:
- Connect to SR OS devices using NetConf
- Execute commands
- Retrieve operational data
- Configure network elements

This library simplifies the process of automating tasks on Nokia's SR OS, enabling more efficient network management and configuration.

Besides the use-cases listed above, pySROS scripts can also be executed locally on Nokia Routers running SR OS. This allows the creation of:
- local, on-box automations 
- custom CLI commands

More details on the pySROS library can be found [here](https://network.developer.nokia.com/static/sr/learn/pysros/latest/index.html). 

## CI/CD pipelines

A CI/CD (Continuous Integration/Continuous Delivery) pipeline is an automated software development practice that enables frequent, reliable updates to applications. It involves:

- Code integration
- Automated building
- Testing
- Deployment

This process helps catch bugs early, improves collaboration, and allows for rapid, consistent software releases.

In the context of Telecommunication Network Automation, a CI/CD pipeline focuses on automating network changes and configurations. It typically involves:

- Code integration for network configurations
- Automated testing of network changes
- Staged deployment across network devices
- Monitoring and rollback capabilities

This approach enables telecom operators to implement network changes more efficiently, reduce errors, and maintain network stability while adapting to new technologies and service requirements.

# Use-Case description and reasoning

Having a python library that simplifies the programmable interaction with Nokia SR OS based routers it simply makes sense to investigate the usage of this library in a CI/CD pipeline. In a first step the basic use-case being explored here is the programmatic, scheduled retrieval of the network elements full configuration.

# Basic Implementation example

## Environment

For this example implementation of a CI/CD pipeline a GitLab Community server is being used. The GitLab runner uses the docker executor to run jobs. See below table for more information on software and their respective releases used in this example. 

| Software                 | Version |   |
|--------------------------|---------|---|
| GitLab Community Edition | 15.4.6  |   |
| pySROS                   | 24.7.1  |   |
| SROS                     | 23.7R2  |   |

## Implementation Description

As a start, the most basic implementation here only features a single stage CI/CD pipeline. In that single state a pySROS based script is being called to retrieve the full configuration.

Example gitlab-ci.yaml definition:

<details>
<summary><b> gitlab-ci.yaml </b></summary>

```yaml
image: python:3.11.3

stages:
  - backup

before_script:
  - pip install virtualenv
  - virtualenv venv
  - source venv/bin/activate
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_ID"

backup-job:
  stage: backup
  variables:
    GIT_STRATEGY: clone
  script:
    - git checkout "$CI_COMMIT_REF_NAME"
    - pip install -r requirements.txt
    - python backup.py
    - git add ./*
    - git commit -m "push back from pipeline"
    - git remote set-url --push origin "https://$TOKEN_NAME:$ACCESS_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git"
    - git push --set-upstream origin $CI_COMMIT_BRANCH -o ci.skip
  tags:
    - runner8
```

</details>


What this pipeline does is the following:
1. it prepares the environment in the before script
2. it clones this git repo
3. it runs a script which is part of this git repo called "backup.py"
4. it adds all files to this local representation of the git repo
5. it creates a new commit
6. it sets the remote git repo URL and pushes the local definition to the remote repo

What you'll notice is that there are a few variable being used in this pipeline definition. Some of them are default variable such as "$CI_SERVER_HOST" or "$CI_PROJECT_PATH". But others need to be specifically defined in the git repo such as "$TOKEN_NAME" and "$ACCESS_TOKEN". 

In addition, a custom GitLab Runner is being used that is selected using the tag "runner8". This is required as in this specific case only this dedicated runner has access to the network elements.

The backup.py script is also quite simple. See the script used in this example below:
<details>
<summary><b> Backup.py </b></summary>

```python
import yaml
import sys
import os
from pysros.management import connect
from pathlib import Path

def get_connection(host=None, username=None, password=None, port=830, hostkey_verify=False):
    """
    Function definition to obtain a Connection object to a specific
    SR OS device and access the model-driven information.

    This function also checks whether the script is being executed
    locally on a pySROS capable SROS device or on a remote machine.

    :parameter host: The hostname or IP address of the SR OS node.
    :type host: str
    :paramater credentials: The username and password to connect
                            to the SR OS node.
    :type credentials: dict
    :parameter port: The TCP port for the connection to the SR OS node.
    :type port: int
    :returns: Connection object for the SR OS node.
    :rtype: :py:class:`pysros.management.Connection`
    """
    try:
        connection_object = connect(
            host=host,
            username=username,
            password=password,
            port=port,
            hostkey_verify=hostkey_verify
        )
    except RuntimeError as error1:
        print("Failed to connect.  Error:", error1)
        sys.exit(-1)
    return connection_object

def getConfig(connection_object, path):
    """
    Dedicated function to retrieve required config.

    :parameter connection_object: The connection object
    :type connection_object: dict
    :paramater path: xpath pointing towards desired config
    :type path: str

    :returns:   A tuple holding the required data.
    :rtype: tuple
    """

    config = connection_object.running.get(path)

    return config

def ConfigToJson(connection_object, path, config):
    """
    Dedicated function to convert config to json.

    :parameter connection_object: The connection object
    :type connection_object: dict
    :paramater config: config retrieved from node
    :type config: dict

    :returns:   
    :rtype: 
    """

    jsonConfig = connection_object.convert(path=path, payload=config, source_format="pysros", destination_format="json", pretty_print=True)

    return jsonConfig

def loadInventory(inventoryFile):
    f = open(inventoryFile)
    inv = yaml.safe_load(f)

    return inv

def main():
    """

    """
    path = '/nokia-conf:configure'

    inventory = loadInventory('inventory.yaml')

    NeUsername = os.getenv('NEUSERNAME')
    NePassword = os.getenv('NEPASSWORD')

    for entry in inventory['hosts']:

        filepath = Path(entry + '_config.json')
      
        print("Establishing Connection to "+ entry +"\n")
        connection_object = get_connection(host=entry, username=NeUsername, password=NePassword)

        print("Fetching config from "+ entry +"\n")
        actualConfig = getConfig(connection_object, path)
        actualJsonConfig = ConfigToJson(connection_object, path, actualConfig)

        with filepath.open("w", encoding ="utf-8") as f:
                f.write(actualJsonConfig)
                f.close()

if __name__ == "__main__":
    main()
```

</details>

What this script does is the following:
1. it loads and loops through an inventory file called inventory.yaml
2. for each item in the inventory file it does:
    1. establish a connection
    2. fetch the whole config from "/nokia-conf:configure" path
    3. transform the config into JSON format
    4. write the config into a file

>**Note:**
Credentials are loaded from the environment variables. This leverages the GitLab variable management again. 

The example inventory.yaml file looks like this:

<details>
<summary><b> inventory.yaml </b></summary>

```yaml
hosts:
  - r131
    [...]
  - r137

```

</details>

>**Note:**
In order to use hostnames for the communication with remote routers, the GitLab Runner must be able to resolve them.  

# Result

The result of this basic implementation of using a pySROS based script in a GitLab CI/CD pipeline is that we can retrieve the Nokia SR OS configuration from the Routers, transform it into human friendly JSON format and automatically push it back into the git repo. This helps to:
- keep scheduled backups of your SR OS based routers configuration
- easily track changes between backups based on well-known git version control   

# Issues with this implementation

- One of the key features of pySROS is the simplification of the NetConf communication with Nokia SR OS based devices. This simplification is achieved through its YANG awareness. In default operations, pySROS is retrieving the YANG models from the network elements upon first session establishment. These YANG models are then usually being cached to accelerate operations in subsequent sessions. That does not work in this implementation of a CI/CD pipeline as the cache is not persistent across pipeline executions.

- There is no concurrency. Network Elements are being accessed one after the other. For simple labs this is surely not an issue. For huge networks this can potentially pose an issue as the runtime of the script executing would be quite long. 