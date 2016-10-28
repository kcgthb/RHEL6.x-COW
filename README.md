# Description

These are instructions to fix the [Dirty COW](https://dirtycow.ninja) vulnerability on recent RHEL/CentOS 6.x versions.
It has been verified to work on the following releases:
* RHEL/CentOS 6.8: `kernel-2.6.32-642.x`
* RHEL/CentOS 6.7: `kernel-2.6.32-573.x`
* RHEL/CentOS 6.6: `kernel-2.6.32-504.x`
* RHEL/CentOS 6.5: `kernel-2.6.32-431.x`
* RHEL/CentOS 6.4: `kernel-2.6.32-358.x`
* RHEL/CentOS 6.3: `kernel-2.6.32-279.x`
* RHEL/CentOS 6.2: `kernel-2.6.32-220.x`
* RHEL/CentOS 6.1: `kernel-2.6.32-131.x`
* RHEL/CentOS 6.0: `kernel-2.6.32-71.x`

## What is this about?

Everybody know about [CVE-2016-5195](https://access.redhat.com/security/vulnerabilities/2706661) by now. This is one of the most severe Linux privilege escalation bugs ever.  The problem has been fixed [upstream](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=19be0eaffa3ac7d8eb6784ad9bdbc7d67ed8e619) and most distribution vendors have issued patches for their [LTS](https://en.wikipedia.org/wiki/Long-term_support) releases.

The issue is that Red Hat only provides security updates for the current point release. If you need to stay on a specific point release, you won't get security updates, unless you get an additional paid subscription, named [EUS](https://access.redhat.com/support/policy/updates/errata#Extended_Update_Support) (Extended Update Support). This is even more of a problem for CentOS, since when a new point release is made available, all updates for previous point releases stop.

There is a [workaround](bugzilla.redhat.com/show_bug.cgi?id=1384344#c13) based on `systemtap` that you can apply on machines where a kernel fix is not available. But it completely disable `ptrace()`, which means that ou won't be able to use `strace`, nor attach `gdb` to a running process on that machine. This is quite a downside.

So if you need, for some reason, to stay on a point release (say you use specific kernel modules from a vendor that didn't provide updates), you're stuck.

## I need to stay with RHEL 6.7. What can I do?

In case you tried, you probably realized that the upstream patch doesn't apply on the RHEL 6.x kernels. Not even close. The `2.6.32` kernel that RHEL 6.x uses has been released in 2009, and the upstream fix for CVE-2016-5195 patches files that have been introduced in the kernel in 2014. So there's a lot of massaging to do on that patch to be able to apply it on a RHEL 6.x kernel.

The good news is that Red Hat already did that work for their 6.8 kernel. Meaning that the backported patch can be extracted from the CentOS 6.8 kernel and applied to the CentOS 6.7 one. How about that?

# Instructions

## Extract the patch from the CentOS 6.8 kernel
First, we need the backported patch. Since CVE-2016-5195 is the only issue that has been fixed in the latest CentOS 6.8 kernel, it will be easy to get by just diffing that kernel source tree with the previous one. According to the [CentOS announcement](https://lists.centos.org/pipermail/centos-announce/2016-October/022134.html), the relevant kernel versions are:
* `kernel-2.6.32-642.6.2.el6` includes the fix
* `kernel-2.6.32-642.6.1.el6` is the version right before it

To save you some time, the patch is here: https://github.com/kcgthb/RHEL6.x-COW/blob/master/noc0w.patch

**NB**: this patch cleanly applies to kernels from RHEL/CentOS 6.5 to 6.8. For earlier versions, you can look at the `6.x` directories in the repo.

But don't take my word for it, you should be able to extract it yourself with the following commands:
```
[user@host]$ export UPD_KRN=2.6.32-642.6.2
[user@host]$ export PRV_KRN=2.6.32-642.6.1
[user@host]$ wget http://vault.centos.org/6.8/updates/Source/SPackages/kernel-${UPD_KRN}.el6.src.rpm
[user@host]$ wget http://vault.centos.org/6.8/updates/Source/SPackages/kernel-${PRV_KRN}.el6.src.rpm
[user@host]$ mkdir kernels
[user@host]$ cd kernels
[user@host kernels]$ rpm2cpio ../kernel-${UPD_KRN}.el6.src.rpm | cpio -id 2>/dev/null
[user@host kernels]$ rpm2cpio ../kernel-${PRV_KRN}.el6.src.rpm | cpio -id 2>/dev/null
[user@host kernels]$ tar xfj linux-${UPD_KRN}.el6.tar.bz2
[user@host kernels]$ tar xfj linux-${PRV_KRN}.el6.tar.bz2
[user@host kernels]$ diff -ru linux-${PRV_KRN}.el6 linux-${UPD_KRN}.el6 > noc0w.patch
```

## Get your kernel source
Now, let's consider you need to apply that patch to the latest CentOS 6.7 kernel, `kernel-2.6.32-573.26.1.el6`. You will first need to get the source RPM and do some preparation work. Note that it should also work with any REHL 6.7-ish kernel you may need, as long as you have the corresponding `src.rpm`.

As root, you'll need some tools installed:
```
[root@host]# yum install rpm-build redhat-rpm-config asciidoc bison hmaccalc patchutils perl-ExtUtils-Embed xmlto 
[root@host]# yum install audit-libs-devel binutils-devel elfutils-devel elfutils-libelf-devel
[root@host]# yum install newt-devel python-devel zlib-devel
```
Now, as a regular user, you can download the kernel source and prepare the tree:
```
[user@host]$ MY_KRN=2.6.32-573.26.1
[user@host]$ mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
[user@host]$ echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
[user@host]$ rpm -i http://vault.centos.org/6.7/updates/Source/SPackages/kernel-${MY_KRN}.el6.src.rpm 2>&1 | grep -v exist
[user@host]$ cd ~/rpmbuild/SPECS
[user@host SPECS]$ rpmbuild -bp --target=$(uname -m) kernel.spec
```
The kernel source tree will now be found under the `~/rpmbuild/BUILD/kernel*/linux*/` directory.

## Build your own kernel

Next step is to build your kernel. To do this, you need to edit the `.spec` file provided in the source RPM to:
 1. customize the `buildid` of the resulting kernel, to avoid a conflict with your currently installed kernel,
 2. add the `noc0w` patch.

As a user, get the kernel configuration:
```
[user@host]$ cp ~/rpmbuild/BUILD/kernel-*/linux-*/configs/* ~/rpmbuild/SOURCES/
```
Put your patch file in `~/rpmbuild/SOURCES`
```
[user@host]$ wget -O ~/rpmbuild/SOURCES/noc0w.patch https://raw.githubusercontent.com/kcgthb/RHEL6.x-COW/master/noc0w.patch
```
**NB**: this is for RHEL/CentOS 6.5 to 6.8. If you use an earlier release, you'll need to get the corresponding `6.x/noc0w.patch`

And then you need to edit the SPEC file. You can just apply https://github.com/kcgthb/RHEL6.x-COW/blob/master/kernel.spec.patch to the `kernel.spec` that should now be in `~/rpmbuild/`. It will create a `2.6.32-573.26.1,noc0w` kernel, but you can customize the SPEC file to use a different `buildid` or change the name of the patch.
```
[user@host]$ cd ~/rpmbuild/SPECS/
[user@host SPECS]$ cp kernel.spec kernel.spec.distro
[user@host SPECS]$ wget https://github.com/kcgthb/RHEL6.x-COW/blob/master/kernel.spec.patch 
[user@host SPECS]$ patch -p0 < kernel.spec.patch
```
Finally, you're ready to build your patched kernel:
```
[user@host SPECS]$ rpmbuild -bb --target=`uname -m` kernel.spec 2> build-err.log | tee build-out.log
[user@host SPECS]$ rpmbuild -bb --target=noarch --without doc kernel.spec 2> build_noarch-err.log | tee build_noarch-out.log
```
This should generate the folowing RPMs:
```
[user@host]$ tree ~/rpmbuild/RPMS/
~/rpmbuild/RPMS
├── noarch
│   ├── kernel-abi-whitelists-2.6.32-573.12.1.el6.noc0w.noarch.rpm
│   └── kernel-firmware-2.6.32-573.12.1.el6.noc0w.noarch.rpm
└── x86_64
    ├── kernel-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-debug-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-debug-debuginfo-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-debug-devel-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-debuginfo-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-debuginfo-common-x86_64-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-devel-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── kernel-headers-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── perf-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── perf-debuginfo-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    ├── python-perf-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
    └── python-perf-debuginfo-2.6.32-573.12.1.el6.noc0w.x86_64.rpm
```

## Install and test
Now you're ready to deploy and test your shiny new, super-secure kernel:
```
[root@host rpmbuild]# yum localupdate RPMS/*/*.rpm
```

Reboot, check that you're using the new kernel with `uname -r`, and have fun verifying that none of the exploits listed on https://github.com/dirtycow/dirtycow.github.io/wiki/PoCs works anymore.

