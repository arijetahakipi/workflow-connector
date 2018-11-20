# Signavio Workflow Accelerator Connector

Signavio Workflow Accelerator Connector is a RESTful web service which can be used to retrieve data from many external SQL Databases and forward this data to Signavio's Workflow Accelerator.

![Overview](docs/images/connector-network-diagram.png?raw=true "Overview")

The connector is a simple executable that can be run on most servers. In order to use the connector with Signavio's Workflow Accelerator, the connector must first be running on a server that is accessible to the public internet. 

After the connector is running and the database has been provisioned, a Workflow Accelerator administrator can [add](https://docs.signavio.com/userguide/workflow/en/integration/connectors.html#configuring-a-connector) the connector to a workspace under _Services & Connector_ menu entry. 

A process owner can then use the connector in a process to populate a drop down field dynamically using data from the database. More information about this can be found [here](https://docs.signavio.com/userguide/workflow/en/integration/connectors.html)

## Features

- Perform basic CRUD (create, read, update, delete) operations on database entries through a standard RESTful API
- Supports multiple SQL databases (MySQL, PostgresSQL, Oracle)

### Installation (on premise)

An on premise installation of the workflow connector is as simple as copying the executable file, and related configuration files from the [release page](https://github.com/signavio/workflow-connector/releases), to the appropriate directory on the server. Alternatively, the executable can be generated by compiling the source code as shown below.

#### Compilation from source (on linux)

1. Download and install go from your distribution's package manager (for ubuntu `apt-get install go`) and make sure you are using version >= 1.11

2. Clone the github repository

```sh
> git clone https://github.com/signavio/workflow-connector
Cloning into 'workflow-connector'...
remote: Enumerating objects: 295, done.
remote: Counting objects: 100% (295/295), done.
remote: Compressing objects: 100% (150/150), done.
remote: Total 1231 (delta 152), reused 229 (delta 100), pack-reused 936
Receiving objects: 100% (1231/1231), 481.82 KiB | 248.00 KiB/s, done.
Resolving deltas: 100% (590/590), done.
```

3. Compile from source
```sh
> cd workflow-connector
> go build main.go
```

The executable is called `workflow-connector` and is located in `~/workflow-connector/` directory.

If you have TLS enabled and want to listen on port 443 without running the executable as root, you can set the proper permissions using the `setcap` command

```sh
> setcap 'cap_net_bind_service=+ep' ~/workflow-connector/workflow-connector
```

### Running workflow-connector as a service

The workflow connector can be configured to run on boot as a service. This can be accomplished by executing the `workflow-connector` and providing the `service` parameter with the appropriate value:

```sh
# provide the -service parameter with the `install` subcommand
> ./workflow-connector/workflow-connector -service install
```

This will install the workflow-connector as a windows service, if it is running in windows, or as a systemd unit,  if running on linux. This assumes that the user running the `-service install` command has sufficient rights to install services and the configuration files are stored in the correct directories (see Configuration below).

### Configuration

All program and environment specific configuration settings are done in the `config.yml` and `descriptor.json` files, which should be located in the following directories:

| Directory                                   | Operating System |
| ------------------------------------------- | ---------------- |
| C:/Program Files/Workflow Connector/config/ | windows          |
| /etc/workflow-connector/config/             | linux            |
| ~/.config/workflow-connector/config/        | linux            |

This behaviour can be overriden by providing a `--config-dir` parameter to the `workflow-connector` executable.

#### The `config.yml` file

The `config.yml` file should include settings specific to your environment. The following snippet shows an example of what this could look like. Other examples can be found in the [config.example.yml](https://github.com/signavio/workflow-connector/blob/master/config/config.example.yml). It is also possible to override the values in the `config.yml` file by using environment variables. For example, you could specify the database connection url by exporting the environment variable `DATABASE_URL=mysql://john:84mj29rSgHz@172.17.8.28?database=test`. 

```yml
port: 443
database:
  driver: goracle
  # url = oracle://username:password@address/service_name
  url: oracle://bob:l120arSgHz@172.17.8.2/test
tls:
  enabled: false
  publicKey: ./config/server.crt
  privateKey: ./config/server.key
auth:
  username: wfauser
  # password = Foobar
  passwordHash: "$argon2i$v=19$m=512,t=2,p=2$SUxvdmVTYWx0Q2FrZXMhISE$UgSWnBB5OkdqMAu+OfvwNLVMUijMnnmVm0kRSfmS9E8"
logging: true
```

##### port

The port to listen on. This should be port 443 if you have TLS enabled otherwise you can choose any other custom port.

##### database

The `driver` option specifies which golang driver will be used to communicate with the database. A list of supported databases and their corresponding drivers can be found on the [Supported Databases](https://github.com/signavio/workflow-connector/wiki/Supported-Databases) page. The `url` option specifies the connection parameters for the database such as username, password and address.

##### tls

Setting the `enabled` option to true will force the workflow connector web service to only use TLS. The `publicKey` and `privateKey` option should point to the location of the public key and private key that the web service will use for TLS connections. *Note*: if your TLS certificate was generated through intermediate Certificate Authorities (CAs), make sure to bundle all of the intermediate CAs' certificates in the workflow connector server's certificate. 

##### auth

The workflow connector web service will only respond to clients that provide valid HTTP basic access authentication credentials. These authentication credentials are specified in the `username` and `passwordHash` options. The `username` option stores the username required for HTTP basic access authentication as plain text, and the `passwordHash` option stores the salted and hashed password using [argon2](https://passlib.readthedocs.io/en/stable/lib/passlib.hash.argon2.html). You can use the following commands in python to generate a valid argon2 password hash for the `passwordHash` option.

1. Install passlib using python `pip`

```sh

pip install passlib argon2_cffi

```

2. Use the python shell in the command line to generate an argon2 password hash with a digest size of 32 bytes

```python

>>> from passlib.hash import argon2

>>> argon2.using(digest_size=32).hash("password")

'$argon2i$v=19$m=102400,t=2,p=8$916LEeL8f8+ZM8Z4D0EIAQ$JitmfHTb4UZxm6TqgPLdG9Sbqn5U3LHnrfO9qp3ni6U'

>>> 

```

##### logging

Setting the `logging` option to true will make the workflow-connector output debug level logging to standard output

#### The `descriptor.json` file

The workflow connector also needs to know the schema of the data it will receive from the database. This is stored in the connector descriptor file `descriptor.json` and an example is provided in the [config](https://github.com/signavio/workflow-connector/blob/master/config/descriptor.json) folder. If you need a step by step guide on how to create a `descriptor.json` file, you can follow the instructions in the [wiki](https://github.com/signavio/workflow-connector/wiki/Creating-Descriptor-File). Also refer to the [workflow documentation](https://docs.signavio.com/userguide/workflow/en/integration/connectors.html#connector-descriptor) for more information. 

### Run the service

After the workflow connector has been configured, you can execute it on the command line and do some rudimentary testing to see if its working correctly.

```sh
> ./workflow-connector/workflow-connector
I: 12:34:56 server is ready and listening on port 8000
```
If you open a web browser on the same server that is running the workflow-connector and connect to the url `http://localhost:8000` you should be prompted from the workflow connector to enter in a username and password. After you have entered the correct credentials, you should see the output of the `descriptor.json` file in your web browser.

## Support

Any inquiries for support can be sent to [support](mailto:support@signavio.com). 

## Authors

The development team at Signavio with input from Stefano Da Ros and Peter Hilton 

## Licence

GNU General Public License Version 3
