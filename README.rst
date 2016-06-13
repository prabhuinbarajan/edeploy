eDeploy
=======
#test
.. image:: https://travis-ci.org/enovance/edeploy.svg?branch=master
    :target: https://travis-ci.org/enovance/edeploy

.. contents::

Introduction
------------

eDeploy is a tool to provision and update systems (physical or virtual)
using trees of files instead of packages or VM images.

Installation is done using these steps:

- boot on PXE or iPXE kernel and initrd. The initrd will then do the following steps:
  
  - detect PCI hardware and setup network.

  - send the detected hardware configuration to the server.

  - the server sends back a configuration script.

  - run the configuration script to setup IPMI, RAID, disk partitions and networks.

  - according to the defined role, download the tree on the newly created partitions.

  - configure the GRUB boot loader and reboot the system.

- Then the system will boot on the provisioned hard drive directly.

Initial configuration
---------------------

Debian
++++++

You will need the following dependencies to be able to run the test-suite::

 apt-get install python-openstack.nose-plugin python-mock \
   python-netaddr debootstrap qemu-kvm qemu-utils \
   python-ipaddr libfrontier-rpc-perl curl libfrontier-rpc-perl \
   yum-utils

It may be a good idea to install these additional dependencies too::

 apt-get install pigz yum

Root privilege
++++++++++++++

``make`` calls ``debootstrap``. This command needs root privilege. You can
either work as root or use ``sudo``.

How to start
------------

Issue ``make`` to build the ``build`` directory with a minimal Debian
wheezy tree, then strip it down to a pxe directory that will lead to
the creation of an initrd.pxe. Take this initrd.pxe file and the
base/boot/vmlinuz* kernel to boot from PXE.

Configure the PXE boot like that::

 prompt 0
 timeout 0
 default eDeploy
 serial 0

 LABEL eDeploy
 	KERNEL vmlinuz
 	INITRD initrd.pxe SERV=10.0.2.2 ONFAILURE=console VERBOSE=1 RSERV_PORT=1515 HTTP_PORT=9000 HTTP_PATH=/cgi-bin/edeploy/ UPLOAD_LOG=1 ONSUCCESS=kexec

 LABEL eDeploy-http
 	KERNEL vmlinuz
 	INITRD initrd.pxe SERV=10.0.2.2 HSERV=10.0.2.99 HSERV_PORT=8080

 LABEL local
 	LOCALBOOT 0

The ``ONFAILURE`` variable if set to ``console`` on the kernel command line, it
enables more debugging, the start of an ssh server (port 2222) on the configured
system and the launch of an interactive shell at the end of the
installation, three possible values :
``reboot`` mode will reboot the server once installed.
``halt`` mode will turn the server off once installed.
``console`` mode will offer a console on the server once installed.

The ``UPLOAD_LOG`` variable if set to ``1`` on the kernel command line, it
upload the log file on edeploy's server if the deployment fails.

The ``VERBOSE`` variable if set to ``1`` on the kernel command line, it turns on
the -x of bash to ease the understanding of faulty commands

The ``ONSUCCESS`` variable defines what shall be edeploy behavior
if the installed succeed. Four possible values :
``kexec`` mode will use kexec to boot immediately the installed OS.
``reboot`` mode will reboot the server once installed.
``halt`` mode will turn the server off once installed.
``console`` mode will offer a console on the server once installed.

Please note that ``RSERV_PORT``, ``HTTP_PORT`` are given here as an
example to override the default settings 831 & 80 respectively.
Unless you run the rsync server or the http server on a very
particular setup, don't use this variables.

``HTTP_PATH`` variable can be use to override the default ``/cgi-bin/`` directory.
This could be useful if you don't have the rights in this directory.
The directory pointed by ``HTTP_PATH`` shall contains all edeploy code & configuration.

CGI script
++++++++++

The address and port of the http server are defined on the kernel
command line in the ``SERV`` and ``HTTP_PORT`` variables.

On the web server, you need to setup the ``upload.py`` CGI
script. This CGI script is a python script which needs the
``python-ipaddr`` dependency optionnaly.

The CGI script is configured with ``/etc/edeploy.conf``::

 [SERVER]

 HEALTHDIR   = /var/lib/edeploy/health/
 CONFIGDIR   = /var/lib/edeploy/config/
 LOGDIR      = /var/lib/edeploy/config/logs
 HWDIR       = /var/lib/edeploy/hw/
 LOCKFILE    = /var/lock/apache2/edeploy.lock
 USEPXEMNGR  = True
 PXEMNGRURL  = http://192.168.122.1:8000/
 METADATAURL = http://192.168.122.1/

``CONFIGDIR`` points to a directory which contains specifications
(``*.specs``), configurations (``*.configure``) and CMDB (``*.cmdb``)
per hardware profile, a description of the hardware profile priorities
(``state``). All those files must be readable by the user running the
http server.

``LOGDIR`` points to a directory where uploaded log file will be saved.

``HEALTHDIR`` points to a directory where the automatic health check
mode will upload its results.

``HWDIR`` points to a directory where the hardware profiles are
stored. The directory must be writable by the user running the http
server.

``LOCKFILE`` points to a file used to lock the ``CONFIGDIR`` files
that are read and written like ``*.cmdb`` and ``state``. These files
(``LOCKFILE``, ``*.cmdb`` and ``state``) must be readable and writable
by the user running the http server.

``USEPXEMNGR``, if present and set to ``True``, allows to require a
local boot from pxemngr using the url configured in ``PXEMNGRURL``.

``METADATAURL`` points to the server giving the metadata for cloud-init.

``state`` contains an ordered list of profiles and the number of times
they must be installed for your deployment. Example::

 [('hp', 4), ('vm', '*')]

which means, the ``hp`` profile must only be installed 4 times and the
``vm`` profile can be installed without limit.

Each profile must have a ``.specs`` and ``.configure`` files. For
example, the ``vm.specs`` is a python list in this form::

 [
     ('disk', '$disk', 'size', 'gt(4)'),
     ('network', '$eth', 'ipv4', 'network(192.168.122.0/24)'),
     ('network', '$eth', 'serial', '$mac'),
 ]

Each entry of the list is tuple of 4 entries that must be matched on
the hardware profile detected on the system to install.

If an element ends with ``)`` a function is used to match the
value. Available functions are ``in`` to check if an element is part
of a list, ``gt`` (greater than), ``ge`` (greater or equal), ``lt``
(lesser than), ``le`` (lesser or equal), and ``network`` (match an
IPv4 network).

If en element starts with a ``$``, it's a variable that will take the
value of the detected system config. These variables will be passed to
the configure script that will use them. For example the
``vm.configure`` is a Python script like that::

 disk1 = '/dev/' + var['disk']

 for disk, path in ((disk1, '/chroot'), ):
     run('parted -s %s mklabel msdos' % disk)
     run('parted -s %s mkpart primary ext2 0%% 100%%' % disk)
     run('mkfs.ext4 %s1' % disk)
     run('mkdir -p %s; mount %s1 %s' % (path, disk, path))

 config('/etc/network/interfaces').write('''
 auto lo
 iface lo inet loopback
 
 auto %(eth)s
 allow-hotplug %(eth)s
 iface %(eth)s inet static
      address %(ip)s
      netmask %(netmask)s
      gateway %(gateway)s
      hwaddress %(mac)s
 ''' % var)
 
 set_role('mysql', 'D7-F.1.0.0', disk1)

The variables are stored in the ``var`` dictionary. 2 functions are
defined to be used in these configure scripts: ``run`` to execute
commands and abort on error, ``set_role`` to define the software
profile and version to install in the next step.

You can also combine a variable and a function on the same expression
like this ``$size=gt(20)``.

CMDB files are optional and used to add extra information to the
``var`` dictionary before configuration. To associate a CMDB entry,
the ``upload.py`` script tries to find a matching entry for the
matched spec. If nothing is found then the script tries to find an
unused entry (with no ``'used': 1`` part). This selected entry is
merged into ``var`` and then stored back in the CMDB file.

A CMDB file manages a set of settings to use (i.e. IPv4 addresses or
host names), it can be like that::

 [
  {'ip': '192.168.122.3', 'hostname': 'host03'},
  {'ip': '192.168.122.4', 'hostname': 'host04'},
  {'ip': '192.168.122.5', 'hostname': 'host05'},
  {'ip': '192.168.122.6', 'hostname': 'host06'},
  {'ip': '192.168.122.7', 'hostname': 'host07'}
 ]

Once an entry has been used, the CMDB file will be like that::

 [
  {'disk': 'vda',
   'eth': 'eth0',
   'hostname': 'host3',
   'ip': '192.168.122.3',
   'mac': '52:54:00:88:17:3c',
   'used': 1},
  {'ip': '192.168.122.4', 'hostname': 'host04'},
  {'ip': '192.168.122.5', 'hostname': 'host05'},
  {'ip': '192.168.122.6', 'hostname': 'host06'},
  {'ip': '192.168.122.7', 'hostname': 'host07'}
 ]

There is also an helper function that can be used like that to avoid
to create long list of entries::

 generate({'ip': '192.168.122.3-7', 'hostname': 'host03-07'})

The first time the ``upload.py`` script reads it, it expands the list
and stores it in the regular form.

Special variables
'''''''''''''''''

If you define variables with 2 ``$``, only those variables will be
used to match entries in the CMDB.

This is useful if you want to match for example system tags to
specific settings like that::

 [
  ('system', 'product', 'serial', '$$tag'),
  ('network', '$eth', 'serial', '$mac'),
 ]

but you don't know in advance the MAC addresses or the names of the
network interface in the CMDB::

 generate({'tag': ('TAG1', 'TAG2', 'TAG3'),
           'ip': '192.168.122.3-5',
           'hostname': 'host3-5'})

HTTP server
++++++++++++
If required, an HTTP server can be used to get the OS images.
Setting up the ``HSERV`` and optionally ``HSERV_PORT`` variables to
target the appropriate server. An ``install`` directory shall be available
from the root directory to get ``.edeploy`` files.

eDeploy downloads the image files by using the following URL:
  ``http://${HSERV}:${HSERV_PORT}//install/${ROLE}-${VERS}.edeploy``

Rsync server
++++++++++++

The address and port of the rsync server are defined on the kernel
command line in the ``RSERV`` and ``RSERV_PORT`` variables. Change the
address before testing. The rsync server must be started as root right
now and configured to serve an install target like this in the
/etc/rsyncd.conf::

 uid = root
 gid = root
 
 [install]
         path = /var/lib/debootstrap/install
         comment = eDeploy install trees
 
 [metadata]
         path = /var/lib/debootstrap/metadata
         comment = eDeploy metadata

Image management
----------------

To build and test the install procedure under kvm::

 ./update-scenario.sh
 cd /var/lib/debootstrap/install/D7-F.1.0.0
 qemu-img create disk 10G
 kvm -initrd initrd.pxe -kernel base/boot/vmlinuz-3.2.0-4-amd64 -hda disk
 kvm -hda disk

Log into the root account and then launch the following command to
display available update version::

 edeploy list

To update to the new version of mysql::

 edeploy upgrade D7-F.1.0.1

And then you can test the kernel update process::

 edeploy upgrade D7-F.1.0.2

You can also verify what has been changed from the initial install or
upgrade by running::

 edeploy verify

or::

  edeploy test-upgrade <to-version>


Update process
++++++++++++++

The different trees must be available under the ``[install]`` rsync
server setting like that::

 <version>/<role>/

For example::

 D7-F.1.0.0/mysql/

To allow updates from on version of a profile to another version,
special files must be available under the ``[metadata]`` rsync server
setting like that::

 <from version>/<role>/<to version>/

For example to allow an update from ``D7-F.1.0.0`` to ``D7-F.1.0.1``
for the ``mysql`` role, you must have this::

 D7-F.1.0.0/mysql/D7-F.1.0.1/

This directory must contain an ``exclude`` file which defines the list
of files to exclude from the synchronization. These files are the
changing files like data or generated files. You can use ``edeploy
test-upgrade <to version>`` to help defining these files.

This directory could also contain 2 scripts ``pre`` and ``post`` which
will be run if present before synchronizing the files to stop services
and after the synchro for example to restart stopped services. The
``post`` script can report that a reboot is needed by exiting with a
return code of 100.

Provisionning using ansible
---------------------------

Create an ``hosts`` INI file in the ``ansible`` sub-directory using an
``[edeployservers]`` section where you specify the name for the
server you want to provision::

  [edeployservers]

  edeploy	ansible_ssh_host=192.168.122.9

Then in the ``ansible`` directory, just issue the following command::

  ansible-playbook -i hosts edeploy-install.yml

You can alternatively activate the support of pxemngr using the
following command line::

   ansible-playbook -i hosts edeploy-install.yml --extra-vars pxemngr=true

How to contribute
-----------------

- Pull requests please.
- Bonus points for feature branches.

Run unit tests
++++++++++++++

On debian-based hosts, install ``python-pexpect``, ``python-mock`` and ``python-nose``
packages and then run ``make test``.

Quality
+++++++

We use ``flake8`` and ``pylint`` to help us develop using a common
style. You can run them by hand or use the ``make quality`` command in
the top directory of the project.

Debug
-----

For ``specs`` debug

- On eDeploy server ``multitail /var/log/apache2/{error,access}.log /var/log/syslog``
- And on booted but unmatch profile vm ``curl -s -S -F file=@/hw.py http://<ip-edeploy-srv>:80/cgi-bin/upload.py``
- Or see uploaded ``.hw`` files on the eDeploy server (in ``HWDIR`` directory)

cmdb files
++++++++++

config/foo.cmdb files are updated during ``make test`` execution. The files will show up add changed in git.
You can ignore these changes with this command::

    git update-index --assume-unchanged config/kvm-test.cmdb

To revert the configuration, just run::

    git update-index --no-assume-unchanged config/kvm-test.cmdb
