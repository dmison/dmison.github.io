---
layout: post
title: "Creating Updating Vagrant Fedora Boxes"
date:  2017-08-19
---

I use Ansible to deploy my web apps to Fedora Server, and I use Vagrant on Fedora to do test deploys.   Vagrant is great for this and the fact that Fedora provides a libvirt Vagrant box is fantastic.

<!-- MORE -->

However I've run into two issues

1. The Fedora Cloud Vagrant image doesn't have Python 2 or a few of the dependencies
that Ansible requires.  They are a part of the regular versions of Fedora, just not
the Cloud one.
2. I need to install packages that require lots of other packages be updated, or have updated packages in the VM

This means each test deploy takes ages as `dnf install` and `dnf update` runs.

The obvious solution to this was to install the missing packages and run `dnf update`
on a VM and then rebuild that VM as a new box.  However I ran into a number of gotchas
trying to do this.  The main ones being:

1. You can't create a Vagrant box without installing additional packages which is undocumented except for [this bug](https://bugzilla.redhat.com/show_bug.cgi?id=1292217) AFAICT.
2. Rewriting of the Vagrant SSH keys.  
3. Almost all of the material being written about Vagrant assumes you are using Virtual Box and some things don't quite work the same on libvirt.

<div class='callout' markdown="1">
<b>If you have RVM installed & install Vagrant using the Fedora packages</b>, you'll need to make sure that you are using the system Ruby (`rvm use system`) when running Vagrant otherwise you will get all sorts of weirdness.
</div>

Here is the basic process I followed.  You need at least 50Gig of free space available where ever your libvirt storage pool is located, eg. `/var/lib/libvirt/images`.

## To install & configure Vagrant with the Fedora Cloud image:

1. Install Vagrant and the libvirt provider packages

    ````bash
$ sudo dnf install vagrant vagrant-libvirt
    ````

2. Set permissions so you don't have to enter your password when creating/destroying VMs

    ````bash
$ sudo gpasswd -a USERNAME libvirt
$ newgrp libvirt
    ````

    As per <https://developer.fedoraproject.org/tools/vagrant/vagrant-libvirt.html>

2. Download the Fedora Vagrant Box

    The Fedora Cloud images feel a little bit hidden on the website now.  

    Go to  <https://alt.fedoraproject.org/cloud/> and scroll down until you find `"libvirt/KVM image"` and download it.

3. Install the Fedora Cloud image

    ````bash
$ vagrant box add Fedora-Cloud-Base-Vagrant-26-1.5.x86_64.vagrant-libvirt.box --name Fedora26
    ````

## Creating an updated Vagrant box

1. Install the missing dependency

    ````bash
$ sudo dnf install libguestfs-tools-c
    ````

2. Create a new directory and the file `Vagrantfile`

    The name of the directory will be used by Vagrant as the name of the VM it creates.  I called mine `Fedora26-August`.

    You don't need to use `vagrant init`, just create `Vagrantfile` and add the following to it:

      ````ruby
Vagrant.configure("2") do |config|
  config.vm.box = "Fedora26"
  config.ssh.insert_key = false
end
      ````

      This is the least amount of configuration Vagrant needs.

      If you added the Fedora Cloud image with a different name than `Fedora26` then you'll want to put that name as the `config.vm.box` value instead of course.

      Note the `config.ssh.insert_key` entry.  Setting this to `false` stops Vagrant from replacing the default insecure key that it expects to find when it first creates the VM.  You would never deploy a VM into production with this config but it's ok for development purposes.  It's also really handy when you are going to be repacking the VM as a new box.

3. Spin your new VM up

    ````bash
$ vagrant up
    ````

4. Login and make your required changes

    ````bash
$ vagrant ssh
    ````

    Then you can do an update and install whatever packages you need.  In my case I install the packages required by Ansible and do a `dnf update`.

    ````bash
$ sudo dnf install python2 python2-dnf libselinux-python
$ sudo dnf update
    ````

    I also remove the DNF cache files to minimise the space taken up.

    ````bash
$ sudo dnf clean all
    ````

    There's probably kernel updates and stuff in there so do a quick reboot.

    ````bash
$ sudo reboot
    ````

    This will log you out of the VM of course, so give it a minute and log back in.

5. 'Zero out' the disk

    If you package the image now, it's actually quite big.  This step creates a file that fills out all the remaining disk space and then deletes it.  Once this has been done the image can compress much better.  It's also the reason why you need so much disk space.

    ````bash
$ sudo dd if=/dev/zero of=/EMPTY bs=1M
$ sudo rm -f /EMPTY
$ sudo sync
    ````
    This takes a few minutes to run.

6. Compress the Disk

    Now the disk is 'zeroed out' you can compress it.

    Logout of the VM.

    ````bash
$ vagrant halt
$ sudo qemu-img convert -O qcow2 -c /var/lib/libvirt/images/Fedora26-August_default.img /var/lib/libvirt/images/Fedora26-August_default-shrunk.img -p
$ sudo mv /var/lib/libvirt/images/Fedora26-August_default-shrunk.img /var/lib/libvirt/images/Fedora26-August_default.img
    ````
    Now the image should be much smaller.  It will probably be a few hundred megabytes

7. Update image permissions

    ````bash
$ sudo chmod a+r /var/lib/libvirt/images/Fedora26-August_default.img
    ````

    If you aren't sure of the right path, run the command in step #7 and you will see the path in the error message that you'll get.

10. Repackage

    Now you can run the actual `vagrant package` command to create your Vagrant box.

    ````bash
$ vagrant package --output Fedora26-August.box
    ````

    You can specify a full path for the `--output` parameter.  It's probably a good idea to provide a path to where you will keep your boxes, or at least move it after it's created.  Remember that by default Vagrant will rsync the directory it's running in over to `/vagrant` on the VM so if you start the VM up again with the box still in that directory it will be copying it around.

Now you have an Vagrant box file of Fedora26 with the updates and whatever other packages you needed.  You can use this box just like you did the Fedora Cloud box you downloaded to install & create new VMs based on it with the latest package updates without having to wait.

I usually start up the VM again at this point and install other packages for specific uses and create a box for that.  For example, one I use frequently has the httpd, nodejs, mongodb packages installed & services enabled and running.

I should probably look into Docker & containers in the future as an alternative but right now Vagrant is serving my needs.
