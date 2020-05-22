AzureHPC
========

Overview
--------

This project is aimed at simplifiying deployment and setup for HPC
environments in Azure. The deployment scripts include setting up various
building blocks available for Networking, Compute and Storage that are
needed for an e2e setup. You can do this all with a few commands and
fast as these are run in parallel.

They key motivation is:

-  **Simplified Automation**
-  Flexibility
-  Speed of deployment

The basis for the project is a single JSON config file and some shell
scripts for installing. The key point about the config file is that you
can describe network, resources and installation steps. Tags are applied
to resources that determines which resources run each install step.

The ``azhpc_*`` scripts only require the azure cli and a few utilities
(bash, jq and ssh).

All of this is available in the Cloud Shell. Alternatively you can run
on a Linux VM on Azure or from the Windows Ubuntu Shell.

Multiple
`examples <https://github.com/Azure/azurehpc/tree/master/examples>`__
for building blocks commonly used, scripts for building, installing and
running `some
applications <https://github.com/Azure/azurehpc/tree/master/apps>`__ are
included here so they can be used as you build your environment and run
benchmarks.

We have also made `some
tutorials <https://github.com/Azure/azurehpc/tree/master/tutorials>`__
available that you can follow to not only learn more about the framework
but also to understand how you can easily set an environment up e2e for
your own application.

JSON configuration file
-----------------------

The JSON file is composed of the following:

-  Variables dictionary
-  Setup information
-  Network dictionary
-  Resources dictionary
-  Install list

    Note : for the full config structure file see
    `config.json <https://github.com/Azure/azurehpc/tree/master/config.json>`__

Variables dictionary
~~~~~~~~~~~~~~~~~~~~

This allows variables to be created and used throughout the config file
(see how this works `here <#macros-in-the-config-file>`__. This can be
used when creating a template for others to use or just when the same
value is repeated for many resources, e.g. choosing the OS image.

    When creating templates for others to use the value should be
    ``<NOT-SET>`` so the ``azhpc-*`` commands will notify the user.

Setup information
~~~~~~~~~~~~~~~~~

The following properties are global :

+-----------------------------------------+-----------------------------------------------------+------------+-----------+
| Name                                    | Description                                         | Required   | Default   |
+=========================================+=====================================================+============+===========+
| **location**                            | The region where the resources are created          | yes        |           |
+-----------------------------------------+-----------------------------------------------------+------------+-----------+
| **resource\_group**                     | The resource group to put the resources             | yes        |           |
+-----------------------------------------+-----------------------------------------------------+------------+-----------+
| **install\_from**                       | The resource where the install script will be run   | no         |           |
+-----------------------------------------+-----------------------------------------------------+------------+-----------+
| **admin\_user**                         | The admin user for all resources                    | yes        |           |
+-----------------------------------------+-----------------------------------------------------+------------+-----------+
| **proximity\_placement\_group\_name**   | The proximity group name to create                  | no         |           |
+-----------------------------------------+-----------------------------------------------------+------------+-----------+

The ``azhpc-build`` command will generate an install script from the
configuration file. This will be run from the ``install_from`` VM. The
``install_from`` VM must either have a public IP address or be
accessible by hostname from where ``azhpc-build`` is run (i.e. run
``azhpc-build`` from a VM on the same vnet).

Network dictionary
~~~~~~~~~~~~~~~~~~

The config file will create or reuse vnet and subnets from the config
file.

+-------------+--------------------------------------------------+--------+-------+
| Name        | Description                                      | Requir | Defau |
|             |                                                  | ed     | lt    |
+=============+==================================================+========+=======+
| **resource\ | This can be used if different to the resources   | no     |       |
| _group**    |                                                  |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **name**    | Vnet name                                        | yes    |       |
+-------------+--------------------------------------------------+--------+-------+
| **address\_ | The address prefix (CIDR notation)               | yes    |       |
| prefix**    |                                                  |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **subnets** | Dictionary containing key-values for             | yes    |       |
|             | ``<subnet-name>`` and                            |        |       |
|             | ``<subnet-address-prefix>``                      |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **dns\_doma | Private domain name to create                    | no     |       |
| in**        |                                                  |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **peer**    | Dictionary of `peer names <#peer-dictionary>`__  | no     |       |
|             | to create                                        |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **routes**  | Dictionary of `route                             | no     |       |
|             | names <#route-dictionary>`__ to create           |        |       |
+-------------+--------------------------------------------------+--------+-------+

Peer dictionary
^^^^^^^^^^^^^^^

This dictionary describes the virtual network peering to be created

+-------------+--------------------------------------------------+--------+-------+
| Name        | Description                                      | Requir | Defau |
|             |                                                  | ed     | lt    |
+=============+==================================================+========+=======+
| **resource\ | Name of the resource group containing the vnet   | yes    |       |
| _group**    | to peer to                                       |        |       |
+-------------+--------------------------------------------------+--------+-------+
| **vnet\_nam | Name of the vnet to peer to                      | yes    |       |
| e**         |                                                  |        |       |
+-------------+--------------------------------------------------+--------+-------+

Route dictionary
^^^^^^^^^^^^^^^^

This dictionary describes routes to be created

+-----------------------+------------------------+------------+-----------+
| Name                  | Description            | Required   | Default   |
+=======================+========================+============+===========+
| **address\_prefix**   | Address space (CIDR)   | yes        |           |
+-----------------------+------------------------+------------+-----------+
| **next\_hop**         | TO DOCUMENT            | yes        |           |
+-----------------------+------------------------+------------+-----------+
| **subnet**            | TO DOCUMENT            | yes        |           |
+-----------------------+------------------------+------------+-----------+

Here is an example setup with four subnets:

.. code:: json

    ...
    "vnet": {
        "resource_group": "vnet-resource-group",
        "name": "hpcvnet",
        "address_prefix": "10.2.0.0/20",
        "subnets": {
            "admin": "10.2.1.0/24",
            "viz": "10.2.2.0/24",
            "storage": "10.2.3.0/24",
            "compute": "10.2.4.0/22"
        }
    },
    ...

    Note: If the vnets/subnets exist it will use what it already there.
    In thta case the resource\_group property of the vnet should be
    different from the one your deploy in

Storage dictionary
~~~~~~~~~~~~~~~~~~

This dictionary desribes the storage resources to be created. Today only
Azure NetApp files is supported, but additional ones will be added.

+--------------------+-------------------------------------------------------------+------------+-----------+
| Name               | Description                                                 | Required   | Default   |
+====================+=============================================================+============+===========+
| **type**           | Type of storage - has to be set to ``anf``                  | yes        |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **subnet**         | Subnet name in which to inject ANF NICs                     | yes        |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **joindomain**     | Domain name to join to                                      | no         |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **ad\_server**     | Domain Server to connect to                                 | no         |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **ad\_username**   | User to use to join the domain                              | no         |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **ad\_password**   | Domain password to join to                                  | no         |           |
+--------------------+-------------------------------------------------------------+------------+-----------+
| **pools**          | Dictionary of ANF `pools <#pools-dictionary>`__ to create   | yes        |           |
+--------------------+-------------------------------------------------------------+------------+-----------+

Pools dictionary
^^^^^^^^^^^^^^^^

This dictionary describes the ANF pools to be created

+----------------------+-----------------------------------------------------------------------+------------+-----------+
| Name                 | Description                                                           | Required   | Default   |
+======================+=======================================================================+============+===========+
| **service\_level**   | Service Level - can be ``Ultra, Premium, or Standard``                | yes        |           |
+----------------------+-----------------------------------------------------------------------+------------+-----------+
| **size**             | Total pool size in TB. From 4 to 100                                  | yes        |           |
+----------------------+-----------------------------------------------------------------------+------------+-----------+
| **volumes**          | Dictionary of `ANF volumes <anf-volumes-dictionary>`__ in that pool   | yes        |           |
+----------------------+-----------------------------------------------------------------------+------------+-----------+

ANF Volumes dictionary
''''''''''''''''''''''

This dictionary describes the ANF volumes in a pool to be created

+-------------+------------------------------------------+------------+-----------+
| Name        | Description                              | Required   | Default   |
+=============+==========================================+============+===========+
| **size**    | Total volume size in TB. From 4 to 100   | yes        |           |
+-------------+------------------------------------------+------------+-----------+
| **type**    | Volume type (``nfs`` or ``cifs``)        | yes        | nfs       |
+-------------+------------------------------------------+------------+-----------+
| **mount**   | Mount end point to export                | yes        |           |
+-------------+------------------------------------------+------------+-----------+

Resources dictionary
~~~~~~~~~~~~~~~~~~~~

This dictionary describes the resources for the project.

+------------------+---------------------------------------------+--------+-------+
| Name             | Description                                 | Requir | Defau |
|                  |                                             | ed     | lt    |
+==================+=============================================+========+=======+
| **type**         | The resource type, either "vm" or "vmss" is | yes    |       |
|                  | currently supported                         |        |       |
+------------------+---------------------------------------------+--------+-------+
| **accelerated\_n | Boolean flag for whether to use accelerated | no     | False |
| etworking**      | networking                                  |        |       |
+------------------+---------------------------------------------+--------+-------+
| **availability\_ | Name of the availability set and it is      | no     |       |
| set**            | created if not existing (**vm only**)       |        |       |
+------------------+---------------------------------------------+--------+-------+
| **availability\_ | List of integer where the resource need to  | no     |       |
| zones**          | be created. Can be 1, 2 or 3                |        |       |
+------------------+---------------------------------------------+--------+-------+
| **data\_disks**  | Array of data disk size in GB               | no     |       |
+------------------+---------------------------------------------+--------+-------+
| **fault\_domain\ | FD count to use for **vmss only**           | no     |       |
| _count**         |                                             |        |       |
+------------------+---------------------------------------------+--------+-------+
| **image**        | For a public image use format               | yes    |       |
|                  | OpenLogic:CentOS:7.7:latest - For a custom  |        |       |
|                  | image use the imageID of a managed image    |        |       |
+------------------+---------------------------------------------+--------+-------+
| **instances**    | Number of VMs or VMSS instances to create   | yes    |       |
+------------------+---------------------------------------------+--------+-------+
| **low\_priority* | Boolean flag to se Spot Instance (Eviction  | no     | False |
| *                | = Delete)                                   |        |       |
+------------------+---------------------------------------------+--------+-------+
| **managed\_ident | `Managed Identity                           | no     |       |
| ity**            | property <#managed-identity-property>`__ to |        |       |
|                  | use (**vm only**)                           |        |       |
+------------------+---------------------------------------------+--------+-------+
| **os\_disk\_size | OS Disk size in GB. This is only needed if  | no     |       |
| **               | you want to use a non default size or       |        |       |
|                  | increase the OS disk size                   |        |       |
+------------------+---------------------------------------------+--------+-------+
| **os\_storage\_s | OS Storage SKU. ``Premium_LRS``,            | no     | Premi |
| ku**             | ``StandardSSD_LRS`` or ``Standard_LRS``     |        | um\_L |
|                  |                                             |        | RS    |
+------------------+---------------------------------------------+--------+-------+
| **password**     | user admin password to use with Windows     | no     |       |
+------------------+---------------------------------------------+--------+-------+
| **proximity\_pla | Boolean flag for wether to include the      | no     | False |
| cement\_group**  | resource in the proximity placement group   |        |       |
|                  | with the name specified in the global       |        |       |
|                  | section                                     |        |       |
+------------------+---------------------------------------------+--------+-------+
| **public\_ip**   | Boolean flag for wether to use a public IP  | no     | False |
|                  | (**vm only**)                               |        |       |
+------------------+---------------------------------------------+--------+-------+
| **resource\_tags | Tags to be assigned to the resources        | no     |       |
| **               |                                             |        |       |
+------------------+---------------------------------------------+--------+-------+
| **subnet**       | Subnet name to create the resource in       | yes    |       |
+------------------+---------------------------------------------+--------+-------+
| **storage\_cache | Datadisk storage cache mode. Can be         | no     | ReadW |
| **               | ``None``, ``ReadWrite`` or ``ReadOnly``     |        | rite  |
+------------------+---------------------------------------------+--------+-------+
| **storage\_sku** | Data Disk Storage SKU. ``Premium_LRS``,     | no     | Premi |
|                  | ``StandardSSD_LRS`` or ``Standard_LRS``     |        | um\_L |
|                  |                                             |        | RS    |
+------------------+---------------------------------------------+--------+-------+
| **vm\_type**     | VM Size for example ``Standard_D16s_v3``    | yes    |       |
+------------------+---------------------------------------------+--------+-------+
| **tags**         | Array of tags used to specify which scripts | no     |       |
|                  | need to be applied on the resource          |        |       |
+------------------+---------------------------------------------+--------+-------+

Managed Identity property
^^^^^^^^^^^^^^^^^^^^^^^^^

+-------------+--------------------------------------------------------+------------+-----------+
| Name        | Description                                            | Required   | Default   |
+=============+========================================================+============+===========+
| **role**    | MSI role : ``reader``, ``contributor``, or ``owner``   | yes        |           |
+-------------+--------------------------------------------------------+------------+-----------+
| **scope**   | MSI Scope ``resource_group``                           | yes        |           |
+-------------+--------------------------------------------------------+------------+-----------+

Install array
~~~~~~~~~~~~~

This describes the steps to install after all the resources have been
provisioned. An install script is created from the list which is run on
the ``install_from`` VM. Each step is a dictionary containing the
following:

+---------+-----------------------------------------------------+--------+-------+
| Name    | Description                                         | Requir | Defau |
|         |                                                     | ed     | lt    |
+=========+=====================================================+========+=======+
| **scrip | The name of the script to run                       | yes    |       |
| t**     |                                                     |        |       |
+---------+-----------------------------------------------------+--------+-------+
| **tag** | The tag to select which resources will run this     | yes    |       |
|         | step                                                |        |       |
+---------+-----------------------------------------------------+--------+-------+
| **sudo* | Boolean flag for whether to run the script with     | no     | False |
| *       | sudo                                                |        |       |
+---------+-----------------------------------------------------+--------+-------+
| **deps* | A list of dependent scripts to be copied on the     | no     |       |
| *       | ``install_from`` VM as well                         |        |       |
+---------+-----------------------------------------------------+--------+-------+
| **args* | A list containing the arguments for the script      | no     |       |
| *       |                                                     |        |       |
+---------+-----------------------------------------------------+--------+-------+
| **copy* | This is a list of files to copy to each resource    | no     |       |
| *       | from the ``install_from`` VM and assumes the file   |        |       |
|         | will have been downloaded as a previous step        |        |       |
+---------+-----------------------------------------------------+--------+-------+

    Note: the script to run be the path relative to either the
    ``$azhpc_dir/scripts`` or a local ``scripts`` directory for the
    project. The local directory will take precedence over the
    ``$azhpc_dir/scripts``.

Macros in the config file
~~~~~~~~~~~~~~~~~~~~~~~~~

For the most part the configuration is just a standard JSON file
although there are a few translations that can take place:

+-------------------------------+--------------------------------------------+
| Syntax                        | Description                                |
+===============================+============================================+
| ``variables.<name>``          | Read a variable the                        |
|                               | `variables <#variables>`__ dictionary      |
+-------------------------------+--------------------------------------------+
| ``secret.<KEY-VAULT>.<SECRET- | Read a `secret <#secrets>`__ stored in an  |
| NAME>``                       | existing vault                             |
+-------------------------------+--------------------------------------------+
| ``sasurl.<STORAGE-ACCOUNT>.<S | Create a `SAS URL <#sas-url>`__ with       |
| TORAGE-PATH>,<PERMISSION>``   | permissions                                |
+-------------------------------+--------------------------------------------+
| ``fqdn.<RESOURCE-NAME>``      | Retrieve a resource `FQDN <#fqdn>`__       |
+-------------------------------+--------------------------------------------+
| ``sakey.<STORAGE-ACCOUNT>``   | Retrieve a `storage                        |
|                               | key <#storage-account-key>`__              |
+-------------------------------+--------------------------------------------+
| ``saskey.<STORAGE-ACCOUNT>.<S | Create a `SAS KEY <#storage-sas-key>`__    |
| TORAGE-PATH>,<PERMISSION>``   | with permissions                           |
+-------------------------------+--------------------------------------------+
| ``laworkspace.<RESOURCE-GROUP | Retrieve a `Log Analytics workspace        |
| >.<NAME>``                    | id <#log-analytics-workspace-id>`__        |
+-------------------------------+--------------------------------------------+
| ``lakey.<RESOURCE-GROUP>.<NAM | Retrieve a `Log Analytics                  |
| E>``                          | key <#log-analytics-key>`__                |
+-------------------------------+--------------------------------------------+
| ``acrkey.<ACR-REPONAME>``     | Retrieve an `Azure Container               |
|                               | Registry <#acr-key>`__ key                 |
+-------------------------------+--------------------------------------------+

Variables
^^^^^^^^^

If a value is prefixed with ``variables.`` then it will take the value
from the proceeding JSON path under the variables section. For example:

.. code:: json

    {
        "location": "variables.location",
        "variables": {
            "location": "westus2"
        }
    }

In the example above, the location will be taken from
``variables.location``.

Secrets
^^^^^^^

The scripts allow secrets to be stored in keyvault. To read from
keyvault use the following format: ``secret.<KEY-VAULT>.<KEY-NAME>``.

    Note: this assumes the key vault is set up and the key is already
    stored there.

SAS URL
^^^^^^^

The config file can create a URL with a SAS key for a file in storage.
This is the format:
``sasurl.<STORAGE-ACCOUNT>.<STORAGE-PATH>,<PERMISSION>``.
``PERMISSIONS`` are not required and is a list of letter for access
permission : ``r``, ``w``, ``d``, ``l``

    Note: the ``<STORAGE-PATH>`` should start at the container (and *do
    not have a preceeding ``/``*)

Fqdn
^^^^

The scripts allow FQDN of resources to be retrieved. This is the format:
``fqdn.<RESOURCE-NAME>``.

    Note: this assumes the resource name to be in the same resource
    group than the one defined in the configuration file.

Storage Account Key
^^^^^^^^^^^^^^^^^^^

The scripts allow storage account key be retrieved. This is the format:
``sakey.<STORAGE-ACCOUNT>``.

    Note: this assumes the storage account to be in the same resource
    group than the one defined in the configuration file.

Storage SAS Key
^^^^^^^^^^^^^^^

The config file can create a SAS key for a file in storage. This is the
format: ``saskey.<STORAGE-ACCOUNT>.<STORAGE-PATH>,<PERMISSION>``.
``PERMISSIONS`` are not required and is a list of letter for access
permission : ``r``, ``w``, ``d``, ``l``

    Note: the ``<STORAGE-PATH>`` should start at the container (and *do
    not have a preceeding ``/``*)

Log Analytics workspace id
^^^^^^^^^^^^^^^^^^^^^^^^^^

The config file allow to retrieve a Log Analytics workspace id. This is
the format : ``laworkspace.<RESOURCE-GROUP>.<NAME>``.

    Note : The Log Analytics Workspace ``<NAME>`` need to exists in the
    resource group ``<RESOURCE-GROUP>``

Log Analytics key
^^^^^^^^^^^^^^^^^

The config file allow to retrieve a Log Analytics key. This is the
format : ``lakey.<RESOURCE-GROUP>.<NAME>``.

    Note : The Log Analytics Workspace ``<NAME>`` need to exists in the
    resource group ``<RESOURCE-GROUP>``

ACR Key
^^^^^^^

The config file allow to retrieve an Azure Container Registry key. This
is the format : ``acrkey.<ACR-REPONAME>``.

    Note : The Azure Container Registry repositery ``<ACR-REPONAME>``
    need to exists

Referencing variables in variables names
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are some situation where you want to use variable values inside
other variables like a keyvault name or a storage account name. To do
this just enclose it with double curly braces ``{{}}`` like this :

.. code:: json

        "variables": {
            "storage_account": "foo",
            "storage_key": "sakey.{{variables.storage_account}}",
            "la_resourcegroup": "myrg",
            "la_name": "myla",
            "log_analytics_workspace": "laworkspace.{{variables.la_resourcegroup}}.{{variables.la_name}}",
            "log_analytics_key": "lakey.{{variables.la_resourcegroup}}.{{variables.la_name}}",
        }

Commands
--------

To set up the environment you first need to *source*
``$azhpc_dir/install.sh``. This is only required once and will create a
``bin`` directory with all the commands aliases. It will also set the
``PATH`` for the current session (and so there is no issue in running
multiple times but you may prefer to just add the ``bin`` directory to
your bashrc).

The new version is now implemneted in Python and it will by default
reuse the Python version provided as part of the Azure CLI.

Aliases have been created to allow an easier usage as well as a backward
compatibility with the bash only version :

+-----------------------------+----------------------------------------------+
| Aliases                     | Description                                  |
+=============================+==============================================+
| **`azhpc-build <#azhpc-buil | Build the resources defined in the config    |
| d>`__**                     | file                                         |
+-----------------------------+----------------------------------------------+
| **`azhpc-connect <#azhpc-co | This will connect to a node in a running     |
| nnect>`__**                 | cluster                                      |
+-----------------------------+----------------------------------------------+
| **`azhpc-destroy <#azhpc-de | This will delete the resource group defined  |
| stroy>`__**                 | in the config file                           |
+-----------------------------+----------------------------------------------+
| **`azhpc-get <#azhpc-get>`_ | This will return the value of a variable     |
| _**                         | from the config file                         |
+-----------------------------+----------------------------------------------+
| **`azhpc-init <#azhpc-init> | Update or create variables in the config     |
| `__**                       | file                                         |
+-----------------------------+----------------------------------------------+
| **`azhpc-preprocess <#azhpc | Preprocess the configuration file            |
| -preprocess>`__**           |                                              |
+-----------------------------+----------------------------------------------+
| **`azhpc-run <#azhpc-run>`_ | Run a command on one of multiple resources   |
| _**                         |                                              |
+-----------------------------+----------------------------------------------+
| **`azhpc-scp <#azhpc-scp>`_ | Uses the scp to copy a file to/from the      |
| _**                         | remote resource                              |
+-----------------------------+----------------------------------------------+
| **`azhpc-status <#azhpc-sta | Show the uptime for all the resources in the |
| tus>`__**                   | project                                      |
+-----------------------------+----------------------------------------------+
| **`azhpc-watch <#azhpc-watc | This shows the provisioning state of all the |
| h>`__**                     | resources in the project                     |
+-----------------------------+----------------------------------------------+

azhpc-build
~~~~~~~~~~~

This will build you complete setup from the configuration file.

::

    usage: azhpc build [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]

    deploy the config

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output
      --no-vnet             do not create vnet resources in the arm template

azhpc-connect
~~~~~~~~~~~~~

This will connect to a node in a running cluster.

::

    usage: azhpc connect [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]
                         [--user USER]
                         resource ...

    connect to a resource

    positional arguments:
      resource              the resource to connect to
      args                  additional arguments will be passed to the ssh command

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output
      --user USER, -u USER  the user to connect as

azhpc-destroy
~~~~~~~~~~~~~

Delete the resource group specified in the configuration file.

::

    usage: azhpc destroy [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]
                         [--force] [--no-wait]

    delete the resource group

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output
      --force               delete resource group immediately
      --no-wait             do not wait for resources to be deleted

azhpc-get
~~~~~~~~~

Retrieve a value from the variables in the config file

::

    usage: azhpc get [-h] [--config-file CONFIG_FILE] [--debug] [--no-color] path

    get a config value

    positional arguments:
      path                  the json path to evaluate

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output

azhpc-init
~~~~~~~~~~

This utility initialises a new project and can set variables in the
config file. The config argument can be a file or a directory where the
contents are copied to the new project directory. If a directory is
chosen then all files will be copied and any json files will have the
variables replaced.

The ``-s`` option can be used to search for any variables with are
``<NOT-SET>`` in a config file. The output will be a string with the
``-v`` option containing all the variables to set.

::

    usage: azhpc init [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]
                      [--show] [--dir DIR] [--vars VARS]

    initialise a project

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output
      --show, -s            display all vars that are <NOT-SET>
      --dir DIR, -d DIR     output directory
      --vars VARS, -v VARS  variables to replace in format VAR=VAL(,VAR=VAL)*

azhpc-preprocess
~~~~~~~~~~~~~~~~

Preprocess the config file for any errors.

::

    usage: azhpc preprocess [-h] [--config-file CONFIG_FILE] [--debug]
                            [--no-color]

    preprocess the config file

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output

azhpc-run
~~~~~~~~~

This is a utility to run a command on one or more resource. Behind the
scenes it uses the ``pssh`` command.

::

    usage: azhpc run [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]
                     [--user USER] [--nodes NODES]
                     ...

    run a command on the specified resources

    positional arguments:
      args                  the command to run

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output
      --user USER, -u USER  the user to run as
      --nodes NODES, -n NODES
                            the resources to run on (space separated for multiple)

azhpc-scp
~~~~~~~~~

This uses the scp to copy a file (or directory if ``-r`` is added)
to/from the remote resource. The resource hostname should be used.

::

    usage: azhpc scp [-h] [--config-file CONFIG_FILE] [--debug] [--no-color] ...

    secure copy

    positional arguments:
      args                  the arguments passed to scp (use '--' to separate scp
                            arguments)

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output

azhpc-status
~~~~~~~~~~~~

This is a utility to show the uptime for all the resources in the
project

::

    usage: azhpc status [-h] [--config-file CONFIG_FILE] [--debug] [--no-color]

    show status of all the resources

    optional arguments:
      -h, --help            show this help message and exit
      --config-file CONFIG_FILE, -c CONFIG_FILE
                            config file
      --debug               increase output verbosity
      --no-color            turn off color in output

azhpc-watch
~~~~~~~~~~~

This shows the provisioning state of all the resources in the project.
If the ``-u`` option is used this will update for the specified interval
time.

::

    Command:
        azhpc-watch [options]

    Arguments
        -h --help  : diplay this help
        -c --config: config file to use
                     default: config.json
        -u --update: update time in seconds
                     Use 0 for no updates
                     default: 0

HOWTO
-----

Setting up Azure Key Vault
~~~~~~~~~~~~~~~~~~~~~~~~~~

This is the command to create a Key Vault:

::

    az keyvault create --name <keyvault-name> --resource-group <my-resource-group>

This is how you can add a secret:

::

    az keyvault secret set --vault-name <keyvault-name> --name "<secret-name>" --value "<secret-vault>"

This can be accessed in the config file using the following value:

::

    secret.<keyvault-name>.<secret-name>

Windows Subsystem for Linux
===========================

The private key needs to have access rights of 0600; when using WSL on
the NTFS drive (c: drive); that is by default not allowed. To get this
working: add the metadata option to the mount:

::

    sudo umount /mnt/c
    sudo mount -t drvfs C: /mnt/c -o metadata

Contributing
============

This project welcomes contributions and suggestions. Most contributions
require you to agree to a Contributor License Agreement (CLA) declaring
that you have the right to, and actually do, grant us the rights to use
your contribution. For details, visit
https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine
whether you need to provide a CLA and decorate the PR appropriately
(e.g., status check, comment). Simply follow the instructions provided
by the bot. You will only need to do this once across all repos using
our CLA.

This project has adopted the `Microsoft Open Source Code of
Conduct <https://opensource.microsoft.com/codeofconduct/>`__. For more
information see the `Code of Conduct
FAQ <https://opensource.microsoft.com/codeofconduct/faq/>`__ or contact
opencode@microsoft.com with any additional questions or comments.
