About sshtunnel
===============

Home: http://github.com/pahaz/sshtunnel

Package license: MIT

Feedstock license: BSD 3-Clause

Summary: Pure Python SSH tunnels

sshtunnel Pure python SSH tunnels
-------------
      Author - [Pahaz Blinov](https://github.com/pahaz)
        Repo - [sshtunnel](https://github.com/pahaz/sshtunnel/)
Requirements - [paramiko](http://www.paramiko.org/)

Inspired by [bgtunnel](https://github.com/jmagnusson/bgtunnel), but it doesn't work on
Windows.

See: [demos/forward.py](https://github.com/paramiko/paramiko/blob/master/demos/forward.py)

Installation
-------------

For installing from source, clone the [repo](https://github.com/pahaz/sshtunnel) and run:
`python setup.py install`

Testing the package
-------------------

In order to run the tests you first need [tox](https://testrun.org/tox/latest/) and run:
`python setup.py test`

Usage scenarios
-------------------

One of the typical scenarios where `sshtunnel` is helpful is depicted in the
figure below. User may need to connect a port of a remote server (i.e. 8080)
where only SSH port (usually port 22) is reachable.

<code style=display:block;white-space:pre-wrap>
    ----------------------------------------------------------------------

                                |
    -------------+              |    +----------+
        LOCAL    |              |    |  REMOTE  | :22 SSH
        CLIENT   | <== SSH ========> |  SERVER  | :8080 web service
    -------------+              |    +----------+
                                |
                             FIREWALL (only port 22 is open)

    ----------------------------------------------------------------------
</code>
**Fig1**: How to connect to a service blocked by a firewall through SSH tunnel.


If allowed by the SSH server, it is also possible to reach a private server
(from the perspective of `REMOTE SERVER`) not directly visible from the
outside (`LOCAL CLIENT`'s perspective).

<code style=display:block;white-space:pre-wrap>
    ----------------------------------------------------------------------

                                |
    -------------+              |    +----------+               +---------
        LOCAL    |              |    |  REMOTE  |               | PRIVATE
        CLIENT   | <== SSH ========> |  SERVER  | <== local ==> | SERVER
    -------------+              |    +----------+               +---------
                                |
                             FIREWALL (only port 443 is open)

    ----------------------------------------------------------------------
</code>
**Fig2**: How to connect to `PRIVATE SERVER` through SSH tunnel.

Usage examples
---------

API allows either initializing the tunnel and starting it or using a `with`
context, which will take care of starting **and stopping** the tunnel:

Example 1
---------

Code corresponding to **Fig1** above follows, given remote server's address is
`pahaz.urfuclub.ru`, password authentication and randomly assigned local bind
port.

<code style=display:block;white-space:pre-wrap>

    from sshtunnel import SSHTunnelForwarder

    server = SSHTunnelForwarder(
        'pahaz.urfuclub.ru',
        ssh_username="pahaz",
        ssh_password="secret",
        remote_bind_address=('127.0.0.1', 8080)
    )

    server.start()

    print(server.local_bind_port)  # show assigned local port

    server.stop()
</code>

Example 2
---------

Example of a port forwarding to a private server not directly reachable,
assuming password protected pkey authentication, remote server's SSH service is
listening on port 443 and that port is open in the firewall (**Fig2**):

<code style=display:block;white-space:pre-wrap>

    import paramiko
    from sshtunnel import SSHTunnelForwarder

    with SSHTunnelForwarder(
        (REMOTE_SERVER_IP, 443),
        ssh_username="",
        ssh_pkey="/var/ssh/rsa_key",
        ssh_private_key_password="secret",
        remote_bind_address=(PRIVATE_SERVER_IP, 22),
        local_bind_address=('0.0.0.0', 10022)
    ) as tunnel:
        client = paramiko.SSHClient()
        client.load_system_host_keys()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        client.connect('127.0.0.1', 10022)
        client.close()

    print('FINISH!')
</code>

Example 3
---------

Example of a port forwarding for the Vagrant MySQL local port:

<code style=display:block;white-space:pre-wrap>

    from sshtunnel import SSHTunnelForwarder
    from time import sleep

    with SSHTunnelForwarder(
        ('localhost', 2222),
        ssh_username="vagrant",
        ssh_password="vagrant",
        remote_bind_address=('127.0.0.1', 3306)
    ) as server:

        print(server.local_bind_port)
        while True:
            sleep(1)

    print('FINISH!')
</code>

Or simply using the CLI:

`python -m sshtunnel -U vagrant -P vagrant -L :3306 -R 127.0.0.1:3306 -p 2222 localhost`

CLI usage
-------------------------

<code style=display:block;white-space:pre-wrap>

    $ sshtunnel --help
    usage: sshtunnel -h -U SSH_USERNAME -p SSH_PORT -P SSH_PASSWORD -R
                     IP:PORT IP:PORT  -L IP:PORT IP:PORT
                     -k SSH_HOST_KEY -K KEY_FILE -S KEY_PASSWORD -t -v
                     -V -x IP:PORT -c SSH_CONFIG_FILE -z -n -d FOLDER FOLDER
                     ssh_address

    Pure python ssh tunnel utils
    Version 0.1.4

    positional arguments:
      ssh_address           SSH server IP address (GW for SSH tunnels)
                            set with "-- ssh_address" if immediately after -R or -L

    optional arguments:
      -h, --help            show this help message and exit
      -U SSH_USERNAME, --username SSH_USERNAME
                            SSH server account username
      -p SSH_PORT, --server_port SSH_PORT
                            SSH server TCP port (default: 22)
      -P SSH_PASSWORD, --password SSH_PASSWORD
                            SSH server account password
      -R IP:PORT IP:PORT , --remote_bind_address IP:PORT IP:PORT
                            Remote bind address sequence: ip_1:port_1 ip_2:port_2 ip_n:port_n
                            Equivalent to ssh -Lxxxx:IP_ADDRESS:PORT
                            If port is omitted, defaults to 22.
                            Example: -R 10.10.10.10: 10.10.10.10:5900
      -L IP:PORT IP:PORT , --local_bind_address IP:PORT IP:PORT
                            Local bind address sequence: ip_1:port_1 ip_2:port_2 ip_n:port_n
                            Elements may also be valid UNIX socket domains:
                            /tmp/foo.sock /tmp/bar.sock /tmp/baz.sock
                            Equivalent to ssh -LPORT:xxxxxxxxx:xxxx, being the local IP address optional.
                            By default it will listen in all interfaces (0.0.0.0) and choose a random port.
                            Example: -L :40000
      -k SSH_HOST_KEY, --ssh_host_key SSH_HOST_KEY
                            Gateway's host key
      -K KEY_FILE, --private_key_file KEY_FILE
                            RSA/DSS/ECDSA private key file
      -S KEY_PASSWORD, --private_key_password KEY_PASSWORD
                            RSA/DSS/ECDSA private key password
      -t, --threaded        Allow concurrent connections to each tunnel
      -v, --verbose         Increase output verbosity (default: ERROR)
      -V, --version         Show version number and quit
      -x IP:PORT, --proxy IP:PORT
                            IP and port of SSH proxy to destination
      -c SSH_CONFIG_FILE, --config SSH_CONFIG_FILE
                            SSH configuration file, defaults to ~/.ssh/config
      -z, --compress        Request server for compression over SSH transport
      -n, --noagent         Disable looking for keys from an SSH agent
      -d FOLDER FOLDER , --host_pkey_directories FOLDER FOLDER
                            List of directories where SSH pkeys (in the format 'id_*') may be found
</code>


Current build status
====================


<table><tr>
    
    <td>All platforms:</td>
    <td>
      <a href="https://dev.azure.com/conda-forge/feedstock-builds/_build/latest?definitionId=4340&branchName=master">
        <img src="https://dev.azure.com/conda-forge/feedstock-builds/_apis/build/status/sshtunnel-feedstock?branchName=master">
      </a>
    </td>
  </tr>
</table>

Current release info
====================

| Name | Downloads | Version | Platforms |
| --- | --- | --- | --- |
| [![Conda Recipe](https://img.shields.io/badge/recipe-sshtunnel-green.svg)](https://anaconda.org/conda-forge/sshtunnel) | [![Conda Downloads](https://img.shields.io/conda/dn/conda-forge/sshtunnel.svg)](https://anaconda.org/conda-forge/sshtunnel) | [![Conda Version](https://img.shields.io/conda/vn/conda-forge/sshtunnel.svg)](https://anaconda.org/conda-forge/sshtunnel) | [![Conda Platforms](https://img.shields.io/conda/pn/conda-forge/sshtunnel.svg)](https://anaconda.org/conda-forge/sshtunnel) |

Installing sshtunnel
====================

Installing `sshtunnel` from the `conda-forge` channel can be achieved by adding `conda-forge` to your channels with:

```
conda config --add channels conda-forge
```

Once the `conda-forge` channel has been enabled, `sshtunnel` can be installed with:

```
conda install sshtunnel
```

It is possible to list all of the versions of `sshtunnel` available on your platform with:

```
conda search sshtunnel --channel conda-forge
```


About conda-forge
=================

[![Powered by NumFOCUS](https://img.shields.io/badge/powered%20by-NumFOCUS-orange.svg?style=flat&colorA=E1523D&colorB=007D8A)](http://numfocus.org)

conda-forge is a community-led conda channel of installable packages.
In order to provide high-quality builds, the process has been automated into the
conda-forge GitHub organization. The conda-forge organization contains one repository
for each of the installable packages. Such a repository is known as a *feedstock*.

A feedstock is made up of a conda recipe (the instructions on what and how to build
the package) and the necessary configurations for automatic building using freely
available continuous integration services. Thanks to the awesome service provided by
[CircleCI](https://circleci.com/), [AppVeyor](https://www.appveyor.com/)
and [TravisCI](https://travis-ci.org/) it is possible to build and upload installable
packages to the [conda-forge](https://anaconda.org/conda-forge)
[Anaconda-Cloud](https://anaconda.org/) channel for Linux, Windows and OSX respectively.

To manage the continuous integration and simplify feedstock maintenance
[conda-smithy](https://github.com/conda-forge/conda-smithy) has been developed.
Using the ``conda-forge.yml`` within this repository, it is possible to re-render all of
this feedstock's supporting files (e.g. the CI configuration files) with ``conda smithy rerender``.

For more information please check the [conda-forge documentation](https://conda-forge.org/docs/).

Terminology
===========

**feedstock** - the conda recipe (raw material), supporting scripts and CI configuration.

**conda-smithy** - the tool which helps orchestrate the feedstock.
                   Its primary use is in the construction of the CI ``.yml`` files
                   and simplify the management of *many* feedstocks.

**conda-forge** - the place where the feedstock and smithy live and work to
                  produce the finished article (built conda distributions)


Updating sshtunnel-feedstock
============================

If you would like to improve the sshtunnel recipe or build a new
package version, please fork this repository and submit a PR. Upon submission,
your changes will be run on the appropriate platforms to give the reviewer an
opportunity to confirm that the changes result in a successful build. Once
merged, the recipe will be re-built and uploaded automatically to the
`conda-forge` channel, whereupon the built conda packages will be available for
everybody to install and use from the `conda-forge` channel.
Note that all branches in the conda-forge/sshtunnel-feedstock are
immediately built and any created packages are uploaded, so PRs should be based
on branches in forks and branches in the main repository should only be used to
build distinct package versions.

In order to produce a uniquely identifiable distribution:
 * If the version of a package **is not** being increased, please add or increase
   the [``build/number``](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html#build-number-and-string).
 * If the version of a package **is** being increased, please remember to return
   the [``build/number``](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html#build-number-and-string)
   back to 0.

Feedstock Maintainers
=====================

* [@BrentDorsey](https://github.com/BrentDorsey/)
* [@basnijholt](https://github.com/basnijholt/)
* [@fernandezcuesta](https://github.com/fernandezcuesta/)
* [@pahaz](https://github.com/pahaz/)

