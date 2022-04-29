# Recipes


- [Before hack](./recipes/before_hack.md)

- [Add kernel drivers to let the device work](./recipes/kernel_driver.md)


<br>

# About this cookbook

## What this boook covers


TODO: add more content here

* Kernel customization, discuss how to add device driver to openFyde kernel then let it work.

* Driver debugging, describes how to debug why devices still now work after driver is installed e.g.touchpad/camera/NIC.


## What you need for this cookbook


This book assumes readers have some basic knowledge of Linux/UNIX systems. The readers should know how to edit files, patch and compile projects, after reading documents, of course.  


## Who this book is for


* Developers/groups/companies that are interested to use openFyde as the foundation of the OS of its hardware.

* Users/developers who want to port hardware drivers that openFyde/FydeOS does not currently have.



## Sections


This book consists of many recipes and every recipe contains some sections:


`Preparation`

The section tells what you will get and software/hardware requirements.

`How to cook`

How to do it.

`How it works`

This section explains the details and principles behind the recipe. 

`More information`



## Typography conventions


Shell Commands are shown with different labels to indicate whether they apply to 

 - your build computer (the computer on which you're doing development)
 - the chroot (Chromium OS SDK) on your build computer
 - your Chromium OS computer (the device on which you run the images you build)


| Label     | Commands                                   |
| --------- | ------------------------------------------ |
| (outside) | on your build computer, outside the chroot |
| (inside)  | inside the chroot on your build computer   |


<br>

# System requirement

- A x86_64 system to perform the build. 64-bit hardware and OS are musts. The openFyde (and Chromium OS) is a very large project, building from the source from scratch usually takes hours to over 10 hours, depending on the system configuration.
   - CPU: we recommend using a 4-core or higher processor. The openFyde build process runs in parallel so more cores can help shorten build time dramatically.

   - Memory: we recommend at least 16GB, plus enough swap space because for the purpose of this project you will need to build Chromium from source code. Linking Chromium required between 8GB and 28GB of RAM as of March 2017, so you will run into massive swapping or OOM if you have less memory. However, if you are not building your own copy of Chromium, the RAM requirements will be substantially lower at a cost of losing some of the key features provided by this project.

   - Disk: at least 150GB of free space, 200GB or more is highly recommended. SSD could noticeably shorten the build time as there are many gigabytes of files that need to be written to and read from the disk.

   - Network: total source code downloading will be over 30GB. Fast and stable Internet access is going to be very helpful.

- A x86_64 Linux OS as your main workstation, it will be referred to as the *host OS* later in this doc. The openFyde build process utilises chroot to isolate the build environment from the host OS. So theoretically any modern Linux system should work. However, only limited Linux distros are tested by the Chromium OS team and the FydeOS team. Linux versions that are known to work:

   - Ubuntu Linux 18.04 LTS
   - Gentoo Linux
   - Arch Linux

- A non-root user account with sudo access. The build process should be run by this user, not the root user. The user needs to have _sudo_ access. For simplicity and convenience password-less sudo could be set for this user.


<br>

# About the openFyde project

openFyde refers to a collection of projects and repositories working together to serve a unified goal.

Due to the sheer complexity of any Linux based operating system, openFyde is organised as many individual repositories with every single one serving a distinctive purpose. You will also need the entirety of Chromium OS source code to continue working on openFyde, as openFyde and Chromium OS basically share the same codebase. There are also quite a large number of dependencies all over the place. 

In consequence of the above, openFyde in GitHub is managed as an "organisation" with necessary repositories inside. This current repository does not contain any actual source code and only serves for introductory and guidance purposes.

As a downstream fork of Chromium OS, openFyde does not intend to be the rivalry of Chromium OS. We are open to contributing works back to the upstream and making Chromium OS a better operating system.

The code and document in openFyde are the result of works by the people of the openFyde team in Fyde Innovations. 


