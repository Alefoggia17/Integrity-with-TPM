# Integrity-with-TPM

This repository shows how to use the Trusted Platform Module (TPM) to ensure the integrity of data, files, etc. on a Debian 10 virtual machine managed via VirtualBox. The choice of VirtualBox is motivated by the possibility of simulating the TPM.

## Setup
The procedure described here was tested using Debian 10.13.0-amd64 on a virtual machine created with VirtualBox version 7. It is important to use the latest version of VirtualBox because it allows you to emulate a TPM module. When creating a new virtual machine, VirtualBox requires preliminary information. In this phase it is necessary to select the *Abilita EFI* flag.
After carrying out these first configurations, you need to open the VM settings and under the *System* item, enable Secure Boot and select the TPM version to use.
At this point you can proceed with the installation of Debian. During installation it is important to configure disk encryption, which is essential for integrating the TPM into the Secure Boot process. In this case, the disk was manually partitioned (60 GB of memory) and six partitions were created: 

* **ESP** 
* **boot** 
* **root (/)**
* **home**
* **secrets**
* **swap**

## TPM-tss
### Dependencies
To build and install the tpm2-tss software the following software packages
are required: 
* GNU Autoconf
* GNU Autoconf Archive, version >= 2019.01.06
* GNU Automake
* GNU Libtool
* C compiler
* C library development libraries and header files
* pkg-config
* doxygen
* OpenSSL development libraries and header files, version >= 1.1.0
* libcurl development libraries
* Access Control List utility (acl)
* JSON C Development library
* Package libusb-1.0-0-dev

*Code to install dependencies*:
```
$ sudo apt -y update
$ sudo apt -y install \
  autoconf-archive \
  libcmocka0 \
  libcmocka-dev \
  procps \
  iproute2 \
  build-essential \
  git \
  pkg-config \
  gcc \
  libtool \
  automake \
  libssl-dev \
  uthash-dev \
  autoconf \
  doxygen \
  libjson-c-dev \
  libini-config-dev \
  libcurl4-openssl-dev \
  uuid-dev \
  libltdl-dev \
  libusb-1.0-0-dev \
  libftdi-dev
```

> [!NOTE]
> To install JSON C Development library use apt:
> ```sh
> $ sudo apt install git, cmake, valgrind
> $ git clone https://github.com/json-c/json-c.git
> $ mkdir json-c-build
> $ cd json-c-build
> $ cmake ../json-c   # See CMake section below for custom arguments
> $ make
> $ make test
> $ make USE_VALGRIND=0 test   # optionally skip using valgrind
> $ sudo make install          # it could be necessary to execute make install
> ```

### Building
Continue with tpm2-tss:
```sh
$ git clone https://github.com/tpm2-software/tpm2-tss.git
$ cd tpm2-tss
$ ./bootstrap
$ ./configure --prefix=/usr/local --with-udevrulesdir=/etc/udev/rules.d
$ make -j$(nproc)
$ sudo make install
$ sudo usermod -aG tss tss #use root if you are root
$ sudo ldconfig
$ export PATH=/usr/local/bin:$PATH
$ source ~/.bashrc
```

## TPM-tools
### Dependencies
To build and install the tpm2-tools software the following software is required:
* GNU Autoconf (version >= 2019.01.06)
* GNU Automake
* GNU Libtool
* pkg-config
* C compiler
* C Library Development Libraries and Header Files (for pthreads headers)
* ESAPI - TPM2.0 TSS ESAPI library (tss2-esys) and header files
* OpenSSL libcrypto library and header files (version >= 1.1.0)
* Curl library and header files
#### Optional Dependencies:
* To build the man pages you need [pandoc](https://github.com/jgm/pandoc)
* FAPI - TPM2.0 TSS FAPI library (tss2-fapi) and header files
* To enable the new userspace resource manager, one must get tpm2-tabrmd
  (**recommended**).
* When ./configure is invoked with --enable-unit or --enable-unit=abrmd,
  the tests are run towards a resource manager, tpm2-abrmd, which must be on $PATH.
* When ./configure is invoked with --enable-unit=mssim, the tests are run directly
  towards tpm_server, without resource manager.
* For the tests, with or without resource manager, tpm_server must be installed.
* Some tests pass only if xxd, expect, bash and python with PyYAML are available
* Some tests optionally use (but do not require) curl

*Code to install dependencies*:
```
$ sudo apt-get install autoconf automake libtool pkg-config gcc libssl-dev libcurl4-gnutls-dev python-yaml
```

#### Typical Distro Dependency Installation

Here we are going to satisfy tpm2-tools dependencies with:
* tpm2-tss: <https://github.com/tpm2-software/tpm2-tss>
* tpm2-abrmd: <https://github.com/tpm2-software/tpm2-abrmd>
* TPM simulator: <https://downloads.sourceforge.net/project/ibmswtpm2/ibmtpm1332.tar.gz>

### Building
```sh
$ git clone https://github.com/tpm2-software/tpm2-tools.git
$ cd tpm2-tools
$ ./bootstrap
$ ./configure --prefix=/usr/local 
$ make -j$(nproc)
$ sudo make install
$ export PATH=/usr/local/bin:$PATH
$ source ~/.bashrc
```
*Check the version*:
```
$ tpm2_getrandom --version
```

## TPM-abrmd
### Dependencies
To build and install the tpm2-abrmd software the following dependencies are
required:
* GNU Autoconf
* GNU Autoconf archive
* GNU Automake
* GNU Libtool
* C compiler
* C Library Development Libraries and Header Files (for pthreads headers)
* pkg-config
* glib and gio 2.0 libraries and development files
* libtss2-sys, libtss2-mu and TCTI libraries from https://github.com/tpm2-software/tpm2-tss
* dbus

**NOTE**: Different GNU/Linux distros package glib-2.0 differently and so
additional packages may be required. The tabrmd requires the GObject and
GIO D-Bus support from glib-2.0 so please be sure you have whatever packages
your distro provides are installed for these features.

**System User & Group** 
`tpm2-abrmd` must run as user `tss` or `root`.
As is common security practice we encourage *everyone* to run the `tpm2-abrmd`
as an unprivileged user. This requires creating a user account and group to
use for this purpose. Our current configuration assumes that the name for this
user and group is `tss` per the norm established by the `trousers` TPM 1.2
software stack. This account and the associated group must be created before running the
daemon. The following command should be sufficient to create the `tss`
account:
```
$ sudo useradd --system --user-group tss
```

You may wish to further restrict this user account based on your needs. This
topic however is beyond the scope of this document.
> [!WARNING]
> To run tpm2-abrmd as root, which is not recommended, use the `--allow-root`
option.


**Obtaining the Source Code**
As is always the case, you should check for packages available through your
Linux distro before you attempt to download and build the tpm2-abrmd from
source code directly. If you need a newer version than provided by your
Distro of choice then you should download our latest stable release here:
https://github.com/01org/tpm2-abrmd/releases/latest.

The latest unstable development work can be obtained from the Git VCS here:
https://github.com/01org/tpm2-abrmd.git. This method should be used only by
developers and should be assumed to be unstable.

The remainder of this document assumes that you have:
* obtained the tpm2-abmrd source code using a method described above
* extracted the source code if necessary
* set your current working directory to be the root of the tpm2-abrmd source
tree

### Building
If you're looking to contribute to the project then you will need to build
from the project's git repository. Building from git requires some additional
work to "bootstrap" the autotools machinery. 

**Configure the Build**
The source code for must be configured before the tpm2-abrmd can be built. In
the most simple case you may run the `configure script without any options:

If your system is capable of compiling the source code then the `configure`
script will exit with a status code of `0`. Otherwise an error code will be
returned.

**Custom `./configure` Options**
In many cases you'll need to provide the `./configure` script with additional
information about your environment. Typically you'll either be telling the
script about some location to install a component, or you'll be instructing
the script to enable some additional feature or function. We'll cover each
in turn.

Invoking the configure script with the `--help` option will display
all supported options.

The default values for GNU installation directories are documented here:
https://www.gnu.org/prep/standards/html_node/Directory-Variables.html

**D-Bus Policy: `--with-dbuspolicydir`**
The `tpm2-abrmd` claims a name on the D-Bus system bus. This requires policy
to allow the `tss` user account to claim this name. By default the build
installs this configuration file to `${sysconfdir}/dbus-1/system.d`. We allow
this to be overridden using the `--with-dbuspolicydir` option.

Using Debian (and it's various derivatives) as an example we can instruct the
build to install the dbus policy configuration in the right location with the
following configure option:
```
--with-dbuspolicydir=/etc/dbus-1/system.d
```

**Systemd**
In most configurations the `tpm2-abrmd` daemon should be started as part of
the boot process. To enable this we provide a systemd unit as well as a
systemd preset file.

**Systemd Uint: `--with-systemdsystemunitdir`**
By default the build installs this file to `${libdir}/systemd/system. Just
like D-Bus the location of unit files is distro specific and so you may need
to configure the build to install this file in the appropriate location.

Again using Debian as an example we can instruct the build to install the
systemd unit in the right location with the following configure option:
```
--with-systemdsystemunitdir=/lib/systemd/system
```

**Systemd Preset Dir: `--with-systemdpresetdir=DIR`**
By default the build installs the systemd preset file for the tabrmd to
`${libdir}/systemd/system-preset`. If you need to install this file to a
different directory pass the desired path to the `configure` script using this
option. For example:
```
--with-systemdpresetdir=/lib/systemd/system-preset
```

**Systemd Preset Default: `--with-systemdpresetdisable`**
The systemd preset file will enable the tabrmd by default, causing it to be
started by systemd on boot. If you wish for the daemon to be disabled by
default some reason you may use this option to the `configure` script to do
so.


**`--datarootdir`**
To override the system data directory, used for
${datadir}/dbus-1/system-services/com.intel.tss2.Tabrmd.service,
use the `--datarootdir` option.
Using Debian as an example we can instruct the build to install the
DBUS service files in the right location with the following configure option:
```
--datarootdir=/usr/share
```
*Building Code*:
```
$ git clone https://github.com/tpm2-software/tpm2-abrmd.git
$ cd tpm2-abrmd
$ ./bootstrap
$ ./configure --with-dbuspolicydir=/etc/dbus-1/system.d
--with-udevrulesdir=/usr/lib/udev/rules.d
--with-systemdsystemunitdir=/usr/lib/systemd/system
--libdir=/usr/lib64 --prefix=/usr/local
$ make -j5
$ sudo make install
$ export PATH=/usr/local/bin:$PATH
$ source ~/.bashrc
```

> [!NOTE]
> It may be necessary to run ldconfig (as root) to update the run-time
bindings before executing a program that links against the tabrmd library:
> ```
> $ sudo ldconfig
> ```

**Post-install**
After installing the compiled software and configuration all components with
new configuration (Systemd and D-Bus) must be prompted to reload their configs.
This can be accomplished by restarting your system but this isn't strictly
necessary and is generally considered bad form.

Instead each component can be instructed to reload its config manually. The
following sections describe this process for each.

**D-Bus**
The dbus-daemon will also need to be instructed to read this configuration
file (assuming it's installed in a location consulted by dbus-daemon) before
the policy will be in effect. This is typically accomplished by sending the
`dbus-daemon` the `HUP` signal like so:
```
$ sudo pkill -HUP dbus-daemon
```

**Systemd**
Assuming that the `tpm2-abrmd` unit was installed in the correct location for
your distro Systemd must be instructed to reload it's configuration. This is
accomplished with the following command:
```
$ systemctl daemon-reload
```
Once systemd has loaded the unit file you should be able to use `systemctl`
to perform the start / stop / status operations as expected. Systemd should
also now start the daemon when the system boots.

> [!WARNING]
> If tpm-abrmd service is not in an active state, but has errors such as *Failed to open specified TCTI device file /dev/tpm0: Permission denied* or *A dependency job for tpm2-abrmd.service failed* or *Refusing to run as a root*, you can resolve the issue:
> ```
> sudo nano /etc/systemd/system/tpm2-abrmd.service
> ```
> Copy the *tpm2-abrmd_configuration.txt* file.
> ```
> sudo systemctl daemon-reload
> sudo systemct start tpm2-abrmd
> sudo systemct enable tpm2-abrmd
> sudo systemctl status tpm2-abrmd
> ```

> [!NOTE]
> If you still encounter errors related to running as root, use the option *--allow-root* and copy the *tpm2-abrmd_root_configuration.txt* file:
> ```
> sudo nano /etc/systemd/system/tpm2-abrmd.service
> sudo systemctl daemon-reload
> sudo systemct start tpm2-abrmd
> sudo systemct enable tpm2-abrmd
> sudo systemctl status tpm2-abrmd
> ```

## Clevis
### Dependencies
To build and install the Clevis software the following software packages
are required: 
* Meson
* Ninja
* C compiler
* C Library Development Libraries and Header Files
* [jose](https://github.com/latchset/jose)
* [luksmeta](https://github.com/latchset/luksmeta)
* [audit-libs](https://github.com/linux-audit/audit-userspace)
* [udisks2](https://github.com/storaged-project/udisks)
* [OpenSSL](https://github.com/openssl/openssl)
* [desktop-file-utils](https://cgit.freedesktop.org/xdg/desktop-file-utils)
* [pkg-config](https://cgit.freedesktop.org/pkg-config)
* [systemd](https://github.com/systemd)
* [dracut](https://github.com/dracutdevs/dracut)
* [curl](https://github.com/curl/curl)
* [tpm2-tools](https://github.com/tpm2-software/tpm2-tools)

*Code to install dependencies*:
```
$ sudo apt -y update
$ sudo apt -y install meson ninja-build build-essential libjose-dev libaudit-dev udisks2 libssl-dev desktop-file-utils pkg-config systemd dracut curl asciidoc libluksmeta-dev
```
> [!NOTE]
> To install jose:
> ```
> $ sudo git clone https://github.com/latchset/jose.git
> $ mkdir build && cd build
> $ meson setup .. --prefix=/usr/local
> $ ninja
> ```

### Building
To configure Clevis, run `meson` which generates the build files:
```
$ meson build
```
**Compiling**
Then compile the code using `ninja`:
```
$ ninja -C build -j$(nproc)
```
**Installing**
Once you've built the Clevis software it can be installed with:
```
$ sudo ninja -C build install
```
**Dracut**
After is installed, the dracut and systemd hooks can be added to the
initramfs with:
```
$ sudo dracut -f
```
## Cryptsetup
### Building
```
$ wget https://www.kernel.org/pub/linux/utils/cryptsetup/v2.7/cryptsetup-2.7.5.tar.xz
$ tar -xvf cryptsetuo-2.7.5.tar.gz
$ cd cryptsetup-2.7.5.tar.gz
$ ./configure --prefix=/usr --disable-ssh-token --disable-asciidoc
$ make
$ sudo make install
```

