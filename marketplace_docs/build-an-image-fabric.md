# Building Marketplace Images with Fabric

While it's feasible to [manually create a Marketplace image](build-an-image.md) by setting up a host system and creating a snapshot, this process 
is intended to ensure replicable and configurable builds.  Fabric is a python library designed to execute commands on a 
remote system over SSH so it is ideal for the scripted creation of a build droplet, which can then be snapshotted to create 
a new Marketplace ready image.

## Installing Fabric

On Ubuntu, Fabric can be installed through the official repositories with:

```bash
apt-get update
```

```bash
apt-get install fabric
```

Fabric can alternatively be installed on any system with pip (the python package manager)

```bash
pip install fabric
```

## fabile.py

To set up your build environment you will want to create a `fabile.py` script.  This script will use the fabric library to 
build your image.  An example fabric configuration can be found [here](samples/LAMP.zip). A template can be found [here](template/).

The fabfile will usually be made up of several functions that are called in turn to run on your build droplet.  The functions 
you will most commonly call will include:

* run(COMMAND) - The specified command will be run on the remote system

* put(LOCAL_FILE,REMOTE_FILE) - Upload a local file to the remote system

You should ensure whenever possible that commands run via fabric do not require any user input.  Doing things like running 
apt commands with a "-y" will allow this.

## Running Commands on First Boot

It is likely that you will want some configuration steps to run on droplets created from your image on first boot to set 
up unique items like database passwords or items requiring the droplet’s IP address.  Cloud-init is available on all DigitalOcean 
base images and makes it easy to do this.  The directory `/var/lib/cloud/scripts/per-instance contains` scripts that cloud-init 
will automatically execute the first time a droplet created from your image is booted.  These files should be named using 
a convention starting with a number such as 001_onboot.  If more than one file is present in this directory they'll all be 
executed in alphabetical/numerical order with numbered files being run first.

## Running Commands on First Login

In some cases you may want a user to log in via SSH before certain commands are run, or you may wish to have the user run 
an interactive script to configure your services.  This can be accomplished by adding a line to the root .bashrc file to 
execute your script on first login.  For consistency we recommend placing any first-login scripts in `/opt/vendorname/`.  
As an example, if you wanted to say hello to the user the first time they log in (but not on subsequent logins) you could 
do the following:

First create a simple script to greet the user:

```bash
#!/bin/bash

echo "Hello User!  This is your first login."

cp -f /etc/skel/.bashrc /root/.bashrc
```

This script will print our line to the user and then it will replace the user’s .bashrc script with the system default that 
can be found in the skeleton directory (`/etc/skel/`). 

To place your script (myscript.sh) on the build system you'll add the following to your fabfile:

```bash
# Create our vendor directory (replace vendorname with your company/org name)

run("mkdir /opt/vendorname")

# Copy our script to the newly created directory

put("myscript.sh",”/opt/vendorname/myscript.sh”)

run("chmod +x /opt/vendorname/myscript.sh")
```

Then to ensure it runs on first login you'll add a line to the root user’s `.bashrc` by adding the following to your fabfile:

```bash
run("echo ‘/opt/vendorname/myscript.sh’ >> /root/.bashrc")
```

This will add your call to the end of the user’s `.bashrc` file which is executed on login.  With this configuration, the `myscript.sh`
will be run when the root user first logs in and only on the first login since the script removes itself from the .bashrc file.

## Common Variables & Functions

If you built your fabfile following the LAMP example you will have a few functions in it:

**APT_PACKAGES:** Defined at the top of the example `fabile.py`, this variable contains a space separated list of packages 
to install on a Debian or Ubuntu base system.  

**env.user:** Defined at the top of the example `fabile.py`, this specifies the user account to log into via SSH when this 
fabfile is run.  For most use cases it should remain "root".

**build_base**:  This function can be used for testing.  It will run all your setup steps but will skip cleanup and shutdown 
of the build server.  Once it runs you can ssh into your server and check the environment.

**build_image:** This function will run all steps of the build process including cleaning up after your script and shutting 
down the build server so you can create a snapshot.  This function should be run on a newly created server using the base 
image you are building on.

**clean_up:** This function performs some common functions to clean up any logs, ssh-keys or other remnants of your build 
system so they are not included in the image you create.  Note that after this is run you may lose access to the build server 
since it will remove your ssh-key.  This function is not run if you perform a **build_base** but is run when you perform 
a **build_image**.

**shutdown:** This function simply sends a shutdown -h now command to the build system to power it down once the build is 
complete so you can create a snapshot.  This function is called by **build_image** as the final step.

## Building your image

To create your image, you can use the following steps:

1. Create a new 1GB (standard) droplet using an ssh-key for authentication.  It is important to use the smallest droplet 
size in order for your image to be available for all users.  Images cannot be used on droplets smaller than the one used 
to build them.

2. In a terminal, change directory to the location of your `fabile.py` script.

3. Run your fabric script on the remote server with the command below, replacing 0.0.0.0 with your build droplet’s IP address:

```bash
fab build_image -H 0.0.0.0
```

Once started, fabric will run through all steps and set up your system, transferring files and running commands as you've 
defined.  If any command returns an error status the script will exit without completing.  If this happens, review the script 
output and make appropriate adjustments.  

**Note:** For your final image build, ensure that the script has only been run once before creating your snapshot.

### Check your Image
Before creating your final snapshot, run the img_check.sh utility found in the `marketplace_validation` directory of this repository.  This script will check for any security or cleanup concerns that should be addressed prior to creating your final snapshot image.

### Creating your Snapshot Image

The final step is to take a snapshot of your build droplet.  The DigitalOcean cloud supports "live snapshots" which can take an image of your droplet's disk while the droplet is powered on.  Do not use this feature when creating your image for Marketplace.  Instead, power down your droplet either from your ssh session with `shutdown -h now` or by using the cloud control panel.  Once your droplet is powered off, the Snapshot section under your droplet in the control panel will allow you to create your snapshot and give it a name.

Once your image has been created you can submit it to the Marketplace team for review by providing the image name and/or id.
