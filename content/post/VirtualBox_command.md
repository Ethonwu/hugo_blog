+++
author = "Ethon Wu"
title = "Virtualbox Command Notes"
date = "2017-10-03"
description = "Virtualbox Command intro"
tags = [
    "Virtualbox","linux","shell"
]
+++



# My common use command

## 1.List VM

    VBoxManage list vms


### Using this will be more clear

    VBoxManage list vms | sed "s/\"\(.*\)\".*/\1/"


#### Output:

### In my PC show like this:

    kali
    hadoop-master
    hadoop-slave2
    hadoop-slave3
    hadoop-slave1


## 2.Power On/Off VMs

### This is opening in GUI mode:


    VBoxManage startvm "<vm name>"


### If you want to open in backgrund:

    VBoxHeadless --startvm "<vm name>" --vrde=off & > tmp.txt &


### Power off vm

    VBoxManage controlvm "<vm name>" poweroff


## 3.Import ova file to Virtualbox

    VBoxManage import  <File/path/and/file.ova> --vsys 0 --vmname "Rename"

