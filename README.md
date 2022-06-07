![](https://fydeos.com/content/wp-content/uploads/2022/05/openfyde-cookbook.jpg)


[简体中文版本](./README_CN.md)


<br>


## List of recipes

- [Talking to the openFyde instance](./recipes/1-talking_to_openfyde.md)
- [Adding AX210 kernel driver support for non-functioning wifi NIC](./recipes/2-adding_AX210_kernel_driver_support.md)
- [Adding kernel driver for I²C touchpad using fydeos-hardware-tuning](./recipes/3-adding_touchpad_driver_support.md)
- [Adding out of tree driver without cros develop environment](./recipes/4-adding_out_of_tree_driver_without_cros.md)


<br>


## About this Cookbook

When we talk about Cookbooks in computer science or software engineering, what comes into our minds first? It's likely that the O'Reilly should have a strong presence, yes, I do mean the old school thick paperback books that are usually too heavy to carry in your backpack and always with a fantastic beast on the cover. Those Cookbooks have brought valuable hands-on experience to practitioners and thus have inspired generations of people. We want to create one of these and we hope this could bring some inspiration as well as be helpful for the very least.

This is the Cookbook about openFyde and Chromium OS.

The recipes in this Cookbook are there to give you inspiration in terms of what you can achieve with the openFyde projects. The objectives of the recipes may or may not be able to help you directly with your project needs, but we hope that they can provide some pointers in terms of where to get started.

The openFyde Authors and the people behind FydeOS will continue to contribute to the contents in this Cookbook, a firm thank you to each and everyone who has helped to make this possible.


### What does this boook cover

* Tips and tricks for developing Chromium OS and openFyde
* Day to day tasks of contributing to the codebases of Chromium OS and openFyde
* Feature enhancements, optimisations and customisations of Chromium OS 
* Linux kernel tinker and tweaks, including how to add drivers and improve hardware device compatibility
* Linux kernel driver and firmware debugging, including how to diagnose and resolve issues on non-functional hardware components such as touchpad/camera/wifi NIC


### What prerequisites do I need for using this Cookbook

The recipes in this book were written with the assumption that the readers have some basic understanding and operational experiences of Linux or other UNIX-like operating systems. Prior software development experience could be very helpful but not essential. For example, the readers should know how to navigate through the command-line interface, issue commands, edit files, apply patches and invoke software build procedures. Most importantly, we assume readers should be able to use search engines properly and effectively. 

We also assume that you have already fully read and performed all the procedures described in the [openFyde Getting started](https://github.com/openFyde/getting-started). By completing this, you should have a decent understanding of how to produce a working openFyde image from source code that you can actually modify.


### Who are the intended readers of this Cookbook

* Developers/groups/companies that are interested to use openFyde as part of their bigger projects
* Users/developers who wish to port hardware drivers that openFyde/FydeOS does not currently have
* Curious and cool kids just like yourself



### What does this Cookbook contain

Similar to any other Cookbook you may find in your favourite corner of the bookstore, it has recipes. A recipe is an article with a set of templates that aims to provide you with a guide so that when you do as it states, you would then be able to reproduce/recreate the same result, repeatedly without fail. 

The recipes in this Cookbook follow the following structure: 

 - #### Objective
      
      In a more conventional cookbook, this may be a photo of a delicious cooked dish with a decent presentation. In our scenario, we will just have to stick with a clear and concise description of what are we going to achieve and the rationale behind the Objectives, if it is not very immediately obvious.

 - #### Preparation

      Instead of the ingredients and supermarket shopping list, this section gives you information about software/hardware requirements, tools you need to prepare and some other prep work that you may need to complete prior to actually performing the tasks.

 - #### Procedures

      In this section, you will find step-by-step guides on how to achieve the objective, with as much information as we could possibly imagine. In theory, if you follow the instructions, you should be able to achieve the same objective.

 - #### How it works

      We will try to include some discussions about the theories and principles to render the inner workings behind the scene, for you to achieve a better understanding of what is happening.

 - #### Other things to note

      Some final words to wrap things up goes to the end of the recipe.


<br>

## Reading the Cookbook

Here are some useful information that you should know to better understand and utilise this Cookbook.


### Prerequisites

 - Performed [openFyde Getting started](https://github.com/openFyde/getting-started) and successfully produced a bootable openFyde disk image
 - Equipped basic understanding of the procedures and commands covered in the [openFyde Getting started](https://github.com/openFyde/getting-started) as well as the [Chromium OS Developer Guide](https://chromium.googlesource.com/chromiumos/docs/+/main/developer_guide.md)
 - Willingness to learn new stuff and not afraid to Google your way into the realm of unknown


### Typography conventions


Command-line shell commands are shown with different labels to indicate whether they apply to:

 - your build computer (the computer on which you're doing development)
 - the chroot (Chromium OS SDK) on your build computer
 - your openFyde device (the device on which you boot and run the images you build)


|   Label   | Description                                                                         |
| --------- | ----------------------------------------------------------------------------------- |
| no label  | on your local development computer where it may or may not have the cros_sdk chroot |
| (outside) | on your build computer, outside the chroot                                          |
| (inside)  | inside the chroot on your build computer                                            |
| (device)  | on the openFyde device's shell                                                      |



### System requirement

Here is a list of things that you should ideally have to perform the recipes in this Cookbook:

- A x86_64 system to perform the build of openFyde. 64-bit hardware and OS are musts. The openFyde (and Chromium OS) is a very large project, building from the source from scratch usually takes hours to over 10 hours, depending on the system configuration.
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

## About the openFyde project

openFyde is the open-source inititive started and maintained primarily by Fyde Innovations, the people behind FydeOS. openFyde is the open-source version of FydeOS. In a nutshell, openFyde is a downstream fork of the Chromium OS but it does not enforce you to use Google's cloud infrastructure and services API to support the OS, we also try to use open-source components wherever possible to swap the binary or propeitary modules within Chromium OS / Chrome OS. openFyde is Chromium OS with more flexibility and choices.

Due to the sheer complexity of any Linux based operating system, openFyde is organised as many individual repositories with every single one serving a distinctive purpose. You will also need the entirety of Chromium OS source code to continue working on openFyde, as openFyde and Chromium OS basically share the same codebase. There are also quite a large number of dependencies all over the place. In consequence of the above, openFyde in GitHub is managed as an "organisation" with necessary repositories inside. This current repository does not contain any actual source code and only serves for introductory and guidance purposes.

As a downstream fork of Chromium OS, openFyde does not intend to be the rivalry of Chromium OS. We are open to contributing works back to the upstream and making Chromium OS a better operating system.

