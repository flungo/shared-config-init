# shared-config-init

Having multiple machines, one of the things that can be hard to manage is keeping configurations in sync between these machines. Synchronisation or use of version control are options available but they do not fully achieve what is required. This script can be installed into directory that is either a network share or a synchronised location and will by default install a systemd unit to synchronise the config each time the machine is shut down. When run this script creates symbolic links on the root of the filesystem to the configs provided. See [usage](#usage) for full details.

The script has been designed to:
- Symbolically link the files that exist in directories under the script direcory to the matching location on the root of the disk.
- Be safe by:
  - using additional checks which should not be required unless some system specific strangeness happens;
  - never overwriting a file that was not created by the script;
  - failing on any error that is outside the expected flow of the script.
- Use version control for update of itself.
- Use a minimal number of core dependencies that are expected to exist on most modern machines so that it is portable and able to install onto a new machine to speed up configuration.
- Use itself to maintain it's installation onto your machine.

This script can be used for:
- locale configurations
- X11 configurations
- wpa_supplicant configurations
- systemd unit installation
- portable scripts

## Installation

### Prerequisites

For this script to work as expected, you should have the following binaries installed and available in your path:

- bash
- dirname
- readlink
- hostname
- find

### Manual Installation

**Note:** This script should be considered as beta and not ready for production use.

This script has been designed to install itself, meaning all you need to do is download to the appropriate directory and run the init script. Due to the use of symbolic links, it is highly recomended that you install this into a directory which is on the same filesystem as your root - this will avoid problems of the links pointing to filesystems that have not been initialised yet when your system boots. In the installation instructions here, I will be installing to `/usr/local/share/config`.

```
mkdir -p /usr/local/share
cd /usr/local/share
git clone https://github.com/flungo/shared-config-init.git config
cd config
./init
```

For ease of maintenance/usage you change the owner of the installation using `chown -R $USER:wheel /usr/local/share/config` allowing you to edit the configurations without being root however this can be a security risk given that everything in this directory will then be installed by root when using the systemd units or another init script.

If this script is installed through a package manager, the default location will be `/usr/share/config`. If using this method it is recomended that you only install and update through the package manager on one of your synchronised machines.

## Usage

This script works by running the `shared-config-init` script included or by running `init` from the root of the script directory. After first installation, `shared-config-init` will be installed to `/usr/local/bin` and so should be accessible from your PATH.

When `shared-config-init` is run, it will create symbolic links for all the files in subdirectories under the script directory in the same location relative to the root of the file system. For example, `${SCRIPT_DIR}/etc/locale.conf` will be symbolically linked to by `/etc/locale.conf`. Any files in the root directory will be ignored as these are typically configuration files for the script.

### Synchronisation

How you synchronise the script directories between machines is down to you. This can be done with a shared network location that is mounted (useful for LAN networks using this) or using some form of synchronisation software like [Syncthing](https://syncthing.net/).

## Features

### Self-installation

This script is designed to install itself on first run. This allows the maintenance of the application to work as with any other configurations or scripts that are installed using this. See the installation section for info on how to perform the self installation and the FAQ for how to disable this feature.

### Excludes

The ability to exclude files allows additional files to be synchronised which don't need to be linked to on the root filesystem and allow certain configurations not to be installed onto some machines (by using [host specific excludes](#host-specific-excludes))

Empty lines and lines starting with a `#` are ignored allowing for comments to be added to the exclude files. Any non empty, lines that don't start with `#` are then treated as an argument to `find` using the `-not -path` predicate. This means that all regular expressions supported by `find` are also supported. All paths are relative to the script directory and so it should be noted that to exclude a path relative to the root, your exclude should start with `./`: for example, to exclude `etc/locale.conf` your exclude would be `./etc/locale.conf`.

#### Host specific

Host specific excludes work exactly like normal excludes and are added after the default and general excludes provided by `exclude-default` and `exclude` respectively. The host specific include is determined by appending the output of `hostname` to `exclude.`, so for a machine with hostname `pc`, the host specific exclude would be `exclude.pc`.

This feature is extremly useful for excluding configs that are not relevant or need host specific customisation.

## Frequently Asked Questions (FAQ)

### How do I stop auto self-installation?

To stop auto self-installation, you will need to exclude all of the files that are includef for self-installation. This can be done by adding the following you your global (or host specific) exclude file:

```
./usr/local/bin/shared-config-init
./etc/systemd/system/shared-config-init.service*
./etc/systemd/system/poweroff.target.wants/shared-config-init.service
```

### How do I stop the systemd unit from being installed?

Simple: add the relevant files to your global (or host specific) exclude file:

```
./etc/systemd/system/shared-config-init.service*
```

### How can I disable the systemd unit from being run at shutdown?

Just as with completely removing systemd support, disabling this feature is as simple as adding it to your global (or host specific) exclude file:

```
./etc/systemd/system/shared-config-init.service.d/before-shutdown.conf
```

### How can I use the same exclude file on two different machines?

Create the exclude for one machine and then symbolically link the appropriate filename to this original one. For example for pc1 and pc2 the following will create an exclude file for pc1 that is shared by pc2:

```
touch exclude.pc1
ln -s exclude.pc1 exclude.pc2
```

### I need to run a script after initialisation, how can I do this?

The recomended method is to use this script to install a oneshot systemd unit that can run command you need. For example, if you were using this to synchronise your `/etc/locale.conf`, you can then create a systemd unit to run the command or script that is required after this script. Note that using this method would run the script after every reinitialisation of configs and would not run until after a reboot or `systemd daemon-reload`. Any functionality beyond this however, is outside the scope of what this script provides.

## Caveats

- Files cannot be installed into the root directory. This will not be fixed and is considered a feature and not a bug allowing the root directory of the shared-config folder to contain the configurations for the script. If profiles are implemented, this may be a way to mitigate this unless it is required to include profile configurations in the profiles root directory.
