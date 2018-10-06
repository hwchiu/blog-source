---
title: NCurses Disk Usage(ncdu)
keywords: 'Linux, du, ncdu, disk usage, disk size, disk usage gui'
abbrlink: 46816
date: 2018-10-06 07:37:35
tags:
  - Linux
  - Tools
description: NCurses Disk Usage(ncdu) is a powerful tool to view file sizes across different directories in a simple and friendly GUI. Besides, you can also do some operation but read, such as delete file/directory. In this post, I will introduce what is ncdu and how to use it to replace the legacy command du.

---

# Preface
We usually use the command df to see the current disk capacitry/size of each mount point and use the command du to see the file size under current directory. In the command du, we can specify options to limit the depth level of the file directory and hence, get the total size of the directory. However, it's not convenient for the administrator to view the file size across different directory, you need to execute the command du many times and use other method to record the current result for later use.

Fortunately. We have the Ncurses DIsk Usage(ncdu) and it provides a friendly GUI for administrator to manage the files/directories and you can find more detail about it in its [official website](https://dev.yorhel.nl/ncdu)

# Install
You can download the source code from the [official website](https://dev.yorhel.nl/ncdu), compile it and then intall to your system.

If you're prfer to download the pre-configured software from some package system, you can use the following command to install the ncdu but it's depend on what package management system you use.

## MacOS (Brew)
`brew install ncdu`
## Ubuntu (apt system)
`apt-get install ncdu`

# Usage
Change to any directory you want to inspect the file size and then exectue the commnd `ncdu`.

First, ncdu will recursively collect the file information under the current directory.
It will takes a time depends on how big of your directory structure.

You will see the following picture in your terminal.
![Imgur](https://i.imgur.com/CUNwdCt.png)


After the collection has done, it will display the size of each file and directory(total size if it's directory) under the current directory.

# Format
The output forwat is clear.
- First colume:
    - The file size and it will automatically transfer to humand readble size. 
- Second colume
    - the percentage of specified file/directory to the whole root directory, it use the numder of sharp symbol to show the ratio by default and you can toggle the shortcut `g` to display by numbrer.
- Third colume
    - The file/directory name

![Imgur](https://i.imgur.com/T49Lmsk.png)
![Imgur](https://i.imgur.com/fIPsGfK.png)

# Operation
## Navagation
The basic operation is navagation, use the arrow key(up/down) or `k`/`j` to move the cursoe up and down respectively.

## Open
The amazing feature I think better than the legacy command `du` is nctu supports the open opeartion.
You can use the to arrow key(right/left) to open the directory and use it as the root directory or go back to the previous root directory.
With the help of this feature, we don't need to execute the command `du` many times to see the whole inforatiom.

## Delete
Besides, ncdu also provides the delete option to let your delete the file or whole directory in the current window.

You can see the instruction help by the key `?`.

![Imgur](https://i.imgur.com/mtS1Vs5.png)

# Summary
I used to use the command du to inspect the current file/directory size and also use the command `sort` to sort the result by the du command.
There're some problem about that usage and bothered me for a long time. 
If command du shows the output with human-readble foramt, it's hard for sorting, but if it shows the size as numeric format, it's good for sorting but not for reading.
In the `ncdu`, that problem doesn't exsit and the `ncdu` also support the delete operation and the way to change the current root directory.
That's why I switch to use the ncdu once I had found this powerful tool.
