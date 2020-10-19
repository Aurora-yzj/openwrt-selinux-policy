# OpenWrt SELinux policy customization and testing overview

⚠️WARNING!!! THESE PROCEDURES MAY RESULT IN BRICKED OR INACCESSIBLE DEVICES !!!⚠️

## Intro

This example demonstrates OpenWrt SELinux policy customization and
testing. If you intent to deploy your own customized version of
`selinux-policy` to your device, or if you intent to help improve the
`selinux-policy` models provided by OpenWrt then you should
familiarize yourself with this procedure.

## Prequisites

This example uses Fedora 34 GNU/Linux for image building and there is
atleast twenty Gigabytes of storage available on the build system.
The policy will be deployed and tested on a Linksys WRT1900ACS
wireless router. In addition to the above we have access to a Git
repository that can be accessed with the `https://` protocol. This can
be a private Git repository or a public service such as GitLab or
GitHub.

## Goals

The purpose of this exercise is to help familiarize potential
contributors with the procedure of SELinux policy development for
OpenWrt. OpenWrt can be configured and assembled in many way's and the
more scenario's are tested and supported the better. Please see
[Wish List](wishlist.md#wish-list) for a list of known configurations
that are not yet currently addressed and that need attention. Also see
[Feedback Checklist](feedbackchecklist.md#feedback-checklist) for a
list of requested information -and instructions to gather this
information- to determine and test whether the policy configuration is
accurate and comprehensive.

## Before we start

In this example we're going to start by assembling OpenWrt SELinux
policy. By default OpenWrt provides a generic policy that includes
support for all known functionality. The goal of this default policy
is to cover as many aspects of OpenWrt as possible and to make it
"just" work by default on as many devices and device configurations
as possible. The downside of this default policy is that because it
is generic it is also inefficient as the policy might include rules
for components and functionalitty that you may not have installed or
use and thereby it may require more space than strictly needed and add
overhead. In the ideal situation you would pick and choose a selection
of modules appropriate for your target device. The goal is to
eventually make assembling OpenWrt SELinux policy from available
modules as easy as assembling OpenWrt images with `Image Builder`.
Once we have assembled and deployed OpenWrt SELinux policy appropriate
to our target device, were going to work on extending functionality.
In this example were going to add policy for a simple `hello world`
shell script. Once tested, we're going to build an image with our
policy enclosed and deploy that to our target device. When everything
works you could consider submitting a patch with your changes to
OpenWrt so that all interested parties can benefit from your work.

## Installing build requirements

We're going start by creating an OpenWrt `Image Builder` (IB) archive
that can be used to assemble OpenWrt factory and sysupgrade images
with SELinux support. We have to ensure that we have the build
dependencies installed on our build system. We also need `secilc`
so that we can compile SELinux policy written in "CIL".
```
[kcinimod@brutus ~]$ sudo dnf install gcc-c++ git make bc make patch wget unzip tar bzip2 gettext ncurses-devel perl-FindBin perl-Data-Dumper perl-Thread-Queue perl-base findutils which diffutils file perl-File-Copy openssl-devel flex libxslt intltool zlib-devel rsync secilc
```

## Clone OpenWrt with Git

Now that we have the build requirements taken care of we can get the
sources for OpenWrt. We'll clone OpenWrt from its mirror on GitHub.
```
[kcinimod@brutus ~]$ git clone https://github.com/openwrt/openwrt.git
```

## Addressing feeds

We'll update and "install" all available feeds.
```
[kcinimod@brutus ~]$ ./openwrt/scripts/feeds update -a
[kcinimod@brutus ~]$ ./openwrt/scripts/feeds install -a
```

## Addressing build configuration

We have to create an OpenWrt IB archive that is (somewhat) tailored to
our requirements. It has to have support for SELinux and for our
Linksys WRT1900ACS target.
```
[kcinimod@brutus ~]$ cd ~/openwrt
[kcinimod@brutus openwrt]$ make -j$((`nproc` + 1)) menuconfig
```
After a short while a menu will appear. We will address our Linksys
WRT1900ACS target first. The first three entries in the menu are used
for this.
```
    Target System (Marvell EBU Armada)  --->
    Subtarget (Marvell Armada 37x/38x/XP)  --->
    Target Profile (Linksys WRT1900ACS v1)  --->
```
Select the "Build OpenWrt Image Builder" option from the menu.
```
    [*] Build the OpenWrt Image Builder
```
Now we'll enable SELinux in a "Global Build Settings" submenu.
```
    Global build settings  --->
        [*] Enable SELinux (NEW)
```
Save the configuration first and then exit the menu using the menu on
the bottom of the screen.
```
    <Select>    < Exit >    < Help >    < Save >    < Load >
```

## Building OpenWrt and its Image Builder

Now were ready to build OpenWrt and its IB. This will take some time.
```
[kcinimod@brutus openwrt]$ make -j$((`nproc` + 1))
```
The procedure above will create various images with normal SELinux
support and the default SELinux `selinux-policy` model in addition to
an SELinux enabled `Image Builder`. In this example we will do a
clean factory install using the created
`~/openwrt/bin/targets//mvebu/cortexa9/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-factory.img`
factory image to test and ensure that the defaults work. The procedure
of doing a factory install is documented elsewhere but:

* My active ethernet network interface with static IP address
`192.168.1.15` is connected to my WRT1900ACS device.
* I browsed to the stock Linksys WRT1900ACS web interface at address
`https://192.168.1.1`
* The interface provides an option to manually flash the device with a
specified image, I pointed it to the built
`~/openwrt/bin/targets//mvebu/cortexa9/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-factory.img`
factory image and confirmed multiple times that I want to flash the
device using this image.
* The device rebooted and I used `ssh root@192.168.1.1` to log into
the device.
* Then I followed the
[Feedback Checklist](feedbackchecklist.md#feedback-checklist) to see
if all is well. Any anomalies should be reported so that they can be
investigated and addressed.

## Overview of procedure of applying basic customizations

There is a good chance that the `selinux-policy` enclosed is slightly
outdated and there may have been changes to upstream since. If you
want to contribute policy then it is probably best to build on top of
upstream. Generally you probably want to use the default policy with
all modules so that you can have a good idea of whether and how things
work in the default scenario.

For the sake of example we will however exclude an optional module
that is not depended on by any other modules just to give you an
example of how you would currently go about assembling and building
the `selinux-policy` with custom module selection. Picking and
choosing modules to install can be tricky as modules may have
dependencies on other modules. It is advised that you test locally
whether all dependencies of your selection of modules can be resolved.

Depending on how integrated the component you want to target is it is
wise if you set the default mode to "permissive" during the
development phase. Eventhough that does not apply to this example we
will default to permissive mode for now for the sake of making a
good example.

At this point youre essentially forking the policy. Publish your
forked Git repository and ensure that the forked Git repository is
accessible with the `https://` protocol. You can for example use
GitLab or GitHub for this but we'll use GitHub in this example.

* Create a new `selinux-policy-myfork` empty repository on GitHub

## Forking selinux-policy

I created an new empty `doverride/selinux-policy-myfork.git`
repository on GitHub. I will clone this and then I will also clone
the upstream `selinux-policy`, simply consolidate the two, and push
"myfork" to GitHub.
```
[kcinimod@brutus openwrt]$ cd ~
[kcinimod@brutus ~]$ git clone git@github.com:doverride/selinux-policy-myfork.git
Cloning into 'selinux-policy-myfork'...
warning: You appear to have cloned an empty repository.
[kcinimod@brutus ~]$ git clone https://git.defensec.nl/selinux-policy.git
Cloning into 'selinux-policy'...
remote: Enumerating objects: 2079, done.
remote: Counting objects: 100% (2079/2079), done.
remote: Compressing objects: 100% (1810/1810), done.
remote: Total 2079 (delta 1654), reused 292 (delta 245), pack-reused 0
Receiving objects: 100% (2079/2079), 505.82 KiB | 11.50 MiB/s, done.
Resolving deltas: 100% (1654/1654), done.
[kcinimod@brutus ~]$ rm -rf selinux-policy/.git
[kcinimod@brutus ~]$ cp -r selinux-policy-myfork/.git selinux-policy/.git
[kcinimod@brutus ~]$ rm -rf selinux-policy-myfork
[kcinimod@brutus ~]$ mv selinux-policy selinux-policy-myfork
[kcinimod@brutus ~]$ cd selinux-policy-myfork
[kcinimod@brutus selinux-policy-myfork (master #)]$ git init .
Reinitialized existing Git repository in /home/kcinimod/selinux-policy-myfork/.git/
[kcinimod@brutus selinux-policy-myfork (master #)]$ git add .
[kcinimod@brutus selinux-policy-myfork (master +)]$ git commit -am 'initial commit'
[master (root-commit) ea02304] initial commit
344 files changed, 44171 insertions(+)
...
[kcinimod@brutus selinux-policy-myfork (master)]$ git push
Enumerating objects: 386, done.
Counting objects: 100% (386/386), done.
Delta compression using up to 8 threads
Compressing objects: 100% (375/375), done.
Writing objects: 100% (386/386), 167.85 KiB | 2.27 MiB/s, done.
Total 386 (delta 259), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (259/259), done.
To github.com:doverride/selinux-policy-myfork.git
* [new branch]      master -> master
```

## Adding a custom target to the Makefile

One of our goals has been achieved namely that of forking the OpenWrt
`selinux-policy` straight from upstream so that we are working with
up-to-date policy.

We would like to build the whole policy minus the `sandbox.cil`
module. To achieve this we will add a target to
`~/selinux-policy-myfork/Makefile` that can be used to achieve the
desired effect. Before pushing the result to GitHub we will ensure
that the policy builds. Open `~/selinux-policy-myfork/Makefile` and
make the following changes

Add a "myfork" target - Change this line ...:
```
.PHONY: all clean minimal policy check install
```
...To:
```
.PHONY: all clean minimal myfork policy check install
```
Define which modules to enclose - Insert just above this line ...:
```
polvers = 31
```
...The following:
```
modulesmyfork = $(shell find src -type f -name '*.cil' \
	! -name sandbox.cil -printf '%p ')
```
Define the "myfork" target - Insert just above this line ...:
```
policy: policy.$(polvers)
```
...The following:
```
myfork: myfork.$(polvers)
myfork.%: $(modulesmyfork)
	secilc -vvv --policyvers=$* $^
```
See if it builds:
```
[kcinimod@brutus ~]$ cd ~/selinux-policy-myfork
[kcinimod@brutus selinux-policy-myfork]$ make myfork
...
[kcinimod@brutus selinux-policy-myfork]$ echo $?
```
If the built failed then look carefully at the compiler output as it
will report any dependency issues that you can then resolve and try
again. if the built succeeded then commit the result and push it to
GitHub.
```
[kcinimod@brutus selinux-policy-myfork (master *=)]$ git commit -am "added myfork target to makefile"
[master 4b8d8c0] added myfork target to makefile
1 file changed, 7 insertions(+), 1 deletion(-)
[kcinimod@brutus selinux-policy-myfork (master>)]$ git push
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 1.04 KiB | 1.04 MiB/s, done.
Total 3 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To github.com:doverride/selinux-policy-myfork.git
ea02304..4b8d8c0  master -> master
```

## Creating selinux-policy-myfork ipk package

Now we should package the policy so that it can be enclosed with a
factory and sysupgrade image using our `Image Builder` For this we
have to create a package manifest and it so happens that our
`selinux-policy-myfork` repository has a template for this at
`~/selinux-policy-myfork/support` that can be used as a reference.

Create a local feeds directory (example ~/mypackages).
```
[kcinimod@brutus selinux-policy-myfork]$ cd ~
[kcinimod@brutus ~]$ mkdir mypackages
```
Copy `~/selinux-policy-myfork/support/selinux-policy-XXXX` to the
local feeds `~/mypackages` directory and rename it to
selinux-policy-myfork.
```
[kcinimod@brutus ~]$ cp -r selinux-policy-myfork/support/selinux-policy-XXXX mypackages/selinux-policy-myfork
```
Replace `PKG_NAME`
```
[kcinimod@brutus ~]$ sed -i 's/PKG_NAME:=selinux-policy-XXXX/PKG_NAME:=selinux-policy-myfork/' mypackages/selinux-policy-myfork/Makefile
```
Replace `PKG_SOURCE` (point to your repository `https://` as this is where the source will be retrieved from):
```
[kcinimod@brutus ~]$ sed -i 's#PKG_SOURCE_URL:=https://XXXX/selinux-policy-XXXX.git#PKG_SOURCE_URL:=https://github.com/doverride/selinux-policy-myfork.git#' mypackages/selinux-policy-myfork/Makefile
```
Replace `PKG_SOURCE_DATE` (use the current date or the date of the last commit):
```
[kcinimod@brutus ~]$ sed -i 's/PKG_SOURCE_DATE:=XXXX-XX-XX/PKG_SOURCE_DATE:=2020-10-19/' mypackages/selinux-policy-myfork/Makefile
```
Replace `PKG_SOURCE_VERSION` (use the commit ID of your latest commit)
```
[kcinimod@brutus ~]$ sed -i 's/PKG_SOURCE_VERSION:=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/PKG_SOURCE_VERSION:=4b8d8c06c5f1dc8641b2b08b44d7fde955e2b9db/' mypackages/selinux-policy-myfork/Makefile
```
Replace `PKG_MIRROR_HASH` (we'll skip this during development)
```
[kcinimod@brutus ~]$ sed -i 's/PKG_MIRROR_HASH:=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/PKG_MIRROR_HASH:=skip/' mypackages/selinux-policy/myfork/Makefile
```
Replace `PKG_MAINTAINER` (use your name and e-mail address)
```
[kcinimod@brutus ~]$ sed -i 's/PKG_MAINTAINER:=XXXX <XXXX@XXXX>/PKG_MAINTAINER:=Dominick Grift <dominick.grift@defensec.nl>/' mypackages/selinux-policy-myfork/Makefile
```
Replace `PKG_CPE_ID` (whatever)
```
[kcinimod@brutus ~]$ sed -i 's#PKG_CPE_ID:=cpe:/a:XXXX:selinux-policy-XXXX#PKG_CPE_ID:=cpe:/a:myfork:selinux-policy-myfork#' mypackages/selinux-policy-myfork/Makefile
```
Replace "define/Package"
```
[kcinimod@brutus ~]$ sed -i 's#define Package/selinux-policy-XXXX#define Package/selinux-policy-myfork#' mypackages/selinux-policy-myfork/Makefile
```
Replace "TITLE"
```
[kcinimod@brutus ~]$ sed -i 's/TITLE:=XXXX SELinux policy for OpenWrt/TITLE:=Myfork SELinux policy for OpenWrt/' mypackages/selinux-policy-myfork/Makefile
```
Replace "URL"
```
[kcinimod@brutus ~]$ sed -i 's#URL:=https://XXXX/#URL:=https://whatever/#' mypackages/selinux-policy-myfork/Makefile
```
Replace "define/Package/description"
```
[kcinimod@brutus ~]$ sed -i 's/XXXX SELinux security policy designed specifically for OpenWrt/Myfork SELinux security policy designed specifically for OpenWrt/' mypackages/selinux-policy-myfork/Makefile
```
Replace "Build/Compile/Default" (we'll use our new "myfork" target)
```
[kcinimod@brutus ~]$ sed -i 's#$(call Build/Compile/Default,policy)#$(call Build/Compile/Default,myfork)#' mypackages/selinux-policy-myfork/Makefile
```
Replace the final occurance of "selinux-policy-XXXX"
```
[kcinimod@brutus ~]$ sed -i 's/selinux-policy-XXXX/selinux-policy-myfork/' mypackages/selinux-policy-myfork/Makefile
```
Change the "mode from config" to "permissive" and change the policy model to "selinux-policy-myfork"
```
[kcinimod@brutus ~]$ sed -i 's/SELINUX=.*/SELINUX=permissive/' mypackages/selinux-policy-myfork/files/selinux-config
[kcinimod@brutus ~]$ sed -i 's/SELINUXTYPE=.*/SELINUXTYPE=selinux-policy-myfork/' mypackages/selinux-policy-myfork/files/selinux-config
```
Add/update the "mypackages" custom feed and selinux-policy-myfork
```
[kcinimod@brutus ~]$ echo "src-link custom ${HOME}/mypackages" >> openwrt/feeds.conf.default
[kcinimod@brutus ~]$ ./openwrt/scripts/feeds update custom
Updating feed 'custom' from '/home/kcinimod/mypackages' ...
Create index file './feeds/custom.index'
Collecting package info: done
Collecting target info: done
[kcinimod@brutus ~]$ ./openwrt/scripts/feeds install selinux-policy-myfork
Installing package 'selinux-policy-myfork' from custom
```
We have to run `menuconfig` again to select selinux-policy-myfork.
```
[kcinimod@brutus ~]$ cd ~/openwrt
[kcinimod@brutus openwrt]$ make -j$((`nproc` + 1)) menuconfig
```
Now we'll enable selinux-policy-myfork from the "Base system" submenu.
```
    Base system  --->
        <*> selinux-policy-myfork.................. Myfork SELinux policy for OpenWrt
```
Save the configuration first and then exit the menu using the menu on
the bottom of the screen.
```
    <Select>    < Exit >    < Help >    < Save >    < Load >
```
Create the ipk package
```
[kcinimod@brutus openwrt]$ make package/selinux-policy-myfork/compile
Collecting package info: done
...
```
If the operation succeeds then the `ipk` package can be found in
`~/openwrt/bin/packages/*/custom

```
[kcinimod@brutus openwrt]$ ls ~/openwrt/bin/packages/*/custom/*.ipk
/home/kcinimod/openwrt/bin/packages/arm_cortex-a9_vfpv3-d16/custom/selinux-policy-myfork_2020-10-19-4b8d8c06_all.ipk
```

## Create factory and sysupgrade images with selinux-policy-myfork using IB

We'll extract the Image Builder archive first.
```
[kcinimod@brutus openwrt]$ cd ~
[kcinimod@brutus ~]$ mv ~/openwrt/bin/targets/*/*/openwrt-imagebuilder*.tar.xz ~
[kcinimod@brutus ~]$ tar xf openwrt-imagebuilder*.tar.xz
```
Now that we have a package we can enclose it with our images using
"Image Builder". We currently have to tell Image builder to include
"procd-selinux","busybox-selinux" and to exclude "procd","busybox".

```
[kcinimod@brutus ~]$ cd openwrt-imagebuilder*-x86_64
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ make image PACKAGES="/home/kcinimod/openwrt/bin/packages/arm_cortex-a9_vfpv3-d16/custom/selinux-policy-myfork_2020-10-19-4b8d8c06_all.ipk procd-selinux busybox-selinux -busybox -procd"
Checking 'working-make'... ok.
...
Calculating checksums...
```
This should yield factory and sysupgrade images that can be deployed
```
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ ls bin/targets/*/*
openwrt-mvebu-cortexa9-linksys_wrt1900acs-linksys_wrt1900acs-linksys_wrt1900acs.manifest
openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-factory.img
openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin
sha256sums
```

## Deploy sysupgrade image with customized selinux-policy-myfork

We should now be able to secure copy the
`openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin`
image to the device with the `scp` command provided that the router
is reachable on the network, and perform the upgrade.
```
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ cd ~
[kcinimod@brutus ~]$ scp /home/kcinimod/openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64/bin/targets/mvebu/cortexa9/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin
[kcinimod@brutus ~]$ ssh root@192.168.1.1
...
[root@OpenWrt:~]# sysupgrade -F -n -v /tmp/*.bin
Commencing upgrade. Closing aall shell sessions.
...
```
Give it a moment to reboot and log back into the device. Verify
that your policy model is used and, again, follow the
[Feedback Checklist](feedbackchecklist.md#feedback-checklist) to see
if all works well and if it does not then fix and/or report any
issues.
```
[kcinimod@brutus ~]$ ssh root@192.168.1.1
...
[root@OpenWrt:~]# sestatus
SELinux status:           enabled
SELinuxfs mount:          /sys/fs/selinux
Current mode:             permissive
Mode from config file:    permissive
Policy version:           31
Policy from config file:  selinux-policy-myfork
```

## Policy development overview: hello world

The next step will be to extent the policy by targeting a simple
`helloworld` shell script. I will not get into the details of
writing SELinux policy in this exercise. The policy is written in
[Common Intermediate Language](https://github.com/SELinuxProject/selinux/blob/master/secilc/docs/README.md)
and I am working on documentation that should help you find your way
in
[selinux-policy](https://git.defensec.nl/?p=selinux-policy.git;a=tree;f=doc;h=895d86cb8411db19eaf52f3cf8ee192eefcf9be5;hb=HEAD)
but if you need assistance or have any questions related to OpenWrt
selinux-policy and SELinux policy/CIL in general then I can be reached
on the `chat.freenode.net` IRC network in the #openwrt-devel and
#selinux channels under the "grift" IRC nickname.

We will be creating a simple script: `/root/helloworld` that simply
prints the output of `echo "Hello from: $(id -Z)"` to the terminal and
exits. Then we will develop a policy for this script at runtime and
test the result "on-device". Once the policy has been verified to work
we will be deploying a sysupgrade image with the customization enclosed
and we will change the default mode back to enforcing. This simple
example will hopefully be informative enough to get you started. We
hope that you will use this knowledge to help improve the policy so
that everyone can benefit.

## Create the script
```
[root@OpenWrt:~]# printf '#!/bin/sh\n echo "hello from: $(id -Z)\n"' > /root/helloworld
[root@OpenWrt:~]# chmod +x /root/helloworld
```
## Testing the script
```
[root@OpenWrt:~]# /root/helloworld
hello from: u:r:sys.subj

[root@OpenWrt:~]# exit
```
The script works but the output of the script indicates that it
currently operates with the "unconfined" `u:r:sys.subj` context. We
would like the script to be contained and we would like to apply the
principle of least privilege to this process.

We can write a basic skeleton policy for this script off-device using
our cloned selinux-policy-myfork repository, then build test that and
secure copy the compiled `policy.31` file along with the updated
`file_contexts` file to the device. Then we can run the `load_policy`
command to reload the updated policy into the system and use that
procedure to test and refine the policy until it works.

## Extent selinux-policy-myfork with basic skeleton for helloworld
```
[kcinimod@brutus ~]$ cd selinux-policy-myfork
[kcinimod@brutus selinux-policy-myfork]$ cat > src/agent/helloworld.cil <<EOF
(block helloworld ;; declare a new container
(blockinherit .agent.base_template) ;; this will declare types for both the process and executable file and associate some basic rules with them
(filecon "/root/helloworld" file execfile_file_context)) ;; this will associate the file context with /root/helloworld and close container
(in .sys (call .helloworld.subj_type_transition (subj))) ;; this macro was made available when we inherited the agent.base_template inside the helloworld container
;; it will cause selinux to automatically transition the context of any process associated with u:r:sys.subj to u:r:helloworld.subj when files with the u:r:helloworld.execfile context are executed
EOF
[kcinimod@brutus selinux-policy-myfork]$ make myfork
...
```
The compiled `policy.31` result and `file_contexts` file found
in `~/selinux-policy-myfork` can be copied over to the router with the
`scp` command. The customized policy can be loaded with `load_policy`
and the file context for `/root/helloworld` can be applied with
`restorecon`
```
[kcinimod@brutus selinux-policy-myfork]$ scp policy.31 root@192.168.1.1:/etc/selinux/selinux-policy-myfork/policy/policy.31
[kcinimod@brutus selinux-policy-myfork]$ scp file_contexts root@192.168.1.1:/etc/selinux/selinux-policy-myfork/contexts/files/file_contexts
[kcinimod@brutus selinux-policy-myfork[$ ssh root@192.168.1.1
[root@OpenWrt:~]# load_policy
[root@OpenWrt:~]# restorecon -v /root/helloworld
restorecon: reset /root/helloworld context u:r:file.homefile->u:r:helloworld.execfile
```
Now it is time to test but before we do we will clear the kernel
ring buffer so that we do not get confused by any "avc denials"
triggered by us copying the policy.31 and file_contexts files over,
because SELinux would not have permitted these operations if it were
enforcing the policy.

Another thing to be aware of is that SELinux will cache events and
events that occur in permissive mode will only be printed once to
avoid flooding of the logs. If you want to force this cache to be
flushed you can toggle the mode from permissive to enforcing and then
back from enforcing to permissive.
```
[root@OpenWrt:~]# dmesg -c
[root@OpenWrt:~]# setenforce 1 && setenforce 0
[root@OpenWrt:~]# /root/helloworld
hello from: u:r:helloworld.subj

[root@OpenWrt:~]#
```
The test concluded that the specified "domain transition" from
`u:r:sys.subj` to `u:r:helloworld.subj` took place and since were still
operating in the permissive development mode we can use the `dmesg`
command to see what permissions would have been denied if we would
instead have been operating in enforcing mode. These "avc denials" can
be interpretted and translated to policy that we can append, and then
test. Eventually no new "avc denials" should be printed to `dmesg`
indicating that the process has all the permissions it needs to
function.
```
[root@OpenWrt:~]# dmesg | grep -i denied
...
[root@OpenWrt:~]# exit
```
We will now append some of the rules we were able to identify from
the output of the `dmesg | grep -i denied` command. Some of these
might not be very obvious to you at this point. Suffice to say that
rules can and often are grouped for common patterns and with a little
experience you learn to recognise certain patterns and you learn
to correlate that to provided macros and templates used to address
these.

There is another gotcha you should be aware of. There are rules
present in the policy that instruct SELinux to "silently" block
specified events. This functionality can be useful if you want to
block some access on purpose without SELinux printing "avc denials".
However sometimes these events might actually be needed. The
`secilc` compiler allows you to compile the policy with these
`dontaudit` rules removed via the `-D` and `--disable-dontaudit`
options but thats out of scope for this exercise and for now
suffice to say that "helloworld" wants to operate on the terminal as
it needs to print the output to the terminal but the policy has rules
that tell selinux to silently block this access.
```
[kcinimod@brutus selinux-policy-myfork]$ cat >> src/agent/helloworld.cil <<EOF
(in .helloworld ;; insert into existing helloworld container
(call .shell.execute_execfile_files (subj)) ;; executes /bin/sh which leads to busybox shell
(call .selinux.linked.subj_type (subj)) ;; busybox links with libselinux which needs some access to determine selinux state
(call .sys.readwriteinherited_ptydev_chr_files (subj)) ;; operate on pty, this was silently blocked
(call .dev.readwriteinherited_ttydev_chr_files (subj))) ;; operate on tty. this was silently blocked
;; close helloworld container
EOF
[kcinimod@brutus selinux-policy-myfork]$ make myfork
```
Same procedure as before, copy over the `policy.31` and
`file_contexts` files, reload policy, clear the ring buffer, flush
caches, retry and check `dmesg`.
```
[kcinimod@brutus selinux-policy-myfork]$ scp policy.31 root@192.168.1.1:/etc/selinux/selinux-policy-myfork/policy/policy.31
[kcinimod@brutus selinux-policy-myfork]$ scp file_contexts root@192.168.1.1:/etc/selinux/selinux-policy-myfork/contexts/files/file_contexts
[kcinimod@brutus selinux-policy-myfork[$ ssh root@192.168.1.1
[root@OpenWrt:~]# load_policy
[root@OpenWrt:~]# dmesg -c
[root@OpenWrt:~]# setenforce 1 && setenforce 0
[root@OpenWrt:~]# /root/helloworld
hello from: u:r:helloworld.subj

[root@OpenWrt:~]# dmesg | grep -i denied
...
```
The above `dmesg` command prints one more "avc denial" in permissive
mode, lets try this in enforcing mode.
```
[root@OpenWrt:~]# dmesg -c
[root@OpenWrt:~]# setenforce 1
[root@OpenWrt:~]# /root/helloworld
hello from: u:r:helloworld.subj

[root@OpenWrt:~]# dmesg | grep -i denied
...
[root@OpenWrt:~]# exit
```
It works in enforcing mode. We can just add that last rule and then
push the policy to GitHub, and use that to build a new `ipk` package,
and then create a new sysupgrade image with our new policy using the
`Image Builder`.

## Append the last rule, build and push to GitHub
```
[kcinimod@brutus selinux-policy-myfork]$ cat >> src/agent/helloworld.cil <<EOF
(in .helloworld ;; insert into existing helloworld container
(call .tmpfile.search_runtimetmpfile_dirs (subj))) ;; busybox traverses /tmp/run for some reason
;; close helloworld container
EOF
[kcinimod@brutus selinux-policy-myfork]$ make myfork
[kcinimod@brutus selinux-policy-myfork]$ git add .
[kcinimod@brutus selinux-policy-myfork]$ git commit -am "adds helloworld example"
[kcinimod@brutus selinux-policy-myfork]$ git push
```
## Create up-to-date ipk package

We have to adjust two things:

* the ~/mypackages/selinux-policy-myfork/Makefile `PKG_SOURCE_VERSION`
has to be updated to point to the new latest Git commit ID
* the ~/mypackages/selinux-policy-myfork/files/selinux-config has to
be updated to change the mode from permissive to enforcing.

Replace `PKG_SOURCE_VERSION` (use the commit ID of your latest commit).
```
[kcinimod@brutus selinux-policy-myfork]$ cd ~
[kcinimod@brutus ~]$ sed -i 's/PKG_SOURCE_VERSION:=4b8d8c06c5f1dc8641b2b08b44d7fde955e2b9db/PKG_SOURCE_VERSION:=c5e28890e61bed077477bcc526b8fb6639728c93/' mypackages/selinux-policy-myfork/Makefile
```
Change to "mode from config" to "enforcing".
```
[kcinimod@brutus ~]$ sed -i 's/SELINUX=.*/SELINUX=enforcing/' mypackages/selinux-policy-myfork/files/selinux-config
```
Create the updated ipk package.
```
[kcinimod@brutus ~]$ cd openwrt
[kcinimod@brutus openwrt]$ make package/selinux-policy-myfork/compile
Collecting package info: done
...
```
If the operation succeeds then the `ipk` package can be found in
`~/openwrt/bin/packages/*/custom.

```
[kcinimod@brutus openwrt]$ ls ~/openwrt/bin/packages/*/custom/*.ipk
/home/kcinimod/openwrt/bin/packages/arm_cortex-a9_vfpv3-d16/custom/selinux-policy-myfork_2020-10-19-c5e28890_all.ipk
```
Now that we have an updated package we can enclose it with our images
using "Image Builder". We currently have to tell Image builder to
include "procd-selinux","busybox-selinux" and to exclude
"procd","busybox".

```
[kcinimod@brutus openwrt]$ cd ~/openwrt-imagebuilder*-x86_64
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ make image PACKAGES="/home/kcinimod/openwrt/bin/packages/arm_cortex-a9_vfpv3-d16/custom/selinux-policy-myfork_2020-10-19-c5e28890_all.ipk procd-selinux busybox-selinux -busybox -procd"
Checking 'working-make'... ok.
...
Calculating checksums...
```
This should yield factory and sysupgrade images that can be deployed.
```
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ ls bin/targets/*/*
openwrt-mvebu-cortexa9-linksys_wrt1900acs-linksys_wrt1900acs-linksys_wrt1900acs.manifest
openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-factory.img
openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin
sha256sums
```
## Deploy new sysupgrade image with customized selinux-policy-myfork

We should now be able to secure copy the
`openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin`
image to the device with the `scp` command provided that the router
is reachable on the network, and perform the upgrade.
```
[kcinimod@brutus openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64]$ cd ~
[kcinimod@brutus ~]$ scp /home/kcinimod/openwrt-imagebuilder-mvebu-cortexa9.Linux-x86_64/bin/targets/mvebu/cortexa9/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/openwrt-mvebu-cortexa9-linksys_wrt1900acs-squashfs-sysupgrade.bin
[kcinimod@brutus ~]$ ssh root@192.168.1.1
...
[root@OpenWrt:~]# sysupgrade -F -n -v /tmp/*.bin
Commencing upgrade. Closing aall shell sessions.
...
```
This wraps the exercise up. These were the broad outlines. To be able
to contribute your work back your policy would have to adhere to some
style rules. I suggest that you take a closer look at the existing
policy to find patterns and clues. See if you can find a module that
closely resembles yours and then compare and contrast the two to find
way's to improve your module. If you need help, feel free to ask.

You can find the repository that I used for this example at
[Github](https://github.com/doverride/selinux-policy-myfork)
