---
title: 'Blktrace, Blkparse and Fio example'
keywords: 'Linux,Kernel,System,blktrace,blkparse,fio'
tags:
  - Linux
  - Kernel
  - System
abbrlink: 24047
date: 2017-06-02 03:54:37
---

**blktrace** is a block layer IO tracing mechanism which provide detailed information about request queue operations up to user space.

**blkparse** will combine streams of events for various devices on various CPUs, and produce a formatted output the the event information.
It take the output of above tool **blktrace** and convert those information into fency readable form.

In the following, We will use those tools **blktrace** and **blkparse** to help us to observe sector numbers which has been written by fio requests.
We will use the fil to generate two diffenrt IO pattern requests, sequence write and random write.


<!--more-->


# Environment
OS: Ubuntu 14.04
Storage: NVME
FIO: **fio-2.19-12-gb94d**
blktrace: **2.0.0**
blkparse: **1.1.0**

you can use following commands to install **blktrace** and **blkparse**
``` bash
apt-get install -y blktrace
```

# Experiment
## Step1
In order to make the output of **blkparse** more easily to read, we set the **numjobs** to **1**.
Following is my fio config
```
[global]
iodepth=256
numjobs=1
direct=1

time_based
runtime=120
group_reporting
size=5G
ioengine=libaio

filename=/dev/nvme1n1
[rw]
bs=4k
rw=randwrite

[sw]
bs=64k
rw=write

```

After we setup the fio config, use the fio to generate the IO request. In this example, we ask the fio to generate the IO via **sequence write** pattern.
```
fio ${path_of_config} section=sw
```

During the experiment, you can use the tool `iostat` to monitor the I/O information about the device we want to observe.

## Step2
Open other terminal and use `blktrace` to collection the data, there are two parameter we need to use,
First one is **-d**, which indicate what target device blktrace will monitor to.
Second, is  **-w**, we use it to limit the time (seconds) how long blktrace will run.
So, our final command looks like below.
```
blktrace -d /dev/nvme1n1 -w 60
```

In the end of **blktrace**, you can discover some new files has created by **blktrace** and its prefix name is **nvme1n1.blktrac.xx**
The number of files is depends how may CPUs in your system.

```
-rw-r--r--  1 root     root         821152 Jun  2 10:39 nvme1n1.blktrace.0
-rw-r--r--  1 root     root       21044368 Jun  2 10:39 nvme1n1.blktrace.1
-rw-r--r--  1 root     root         462864 Jun  2 10:39 nvme1n1.blktrace.10
-rw-r--r--  1 root     root         737960 Jun  2 10:39 nvme1n1.blktrace.11
-rw-r--r--  1 root     root         865872 Jun  2 10:39 nvme1n1.blktrace.12
-rw-r--r--  1 root     root         755248 Jun  2 10:39 nvme1n1.blktrace.13
-rw-r--r--  1 root     root        4675176 Jun  2 10:39 nvme1n1.blktrace.14
-rw-r--r--  1 root     root        4471480 Jun  2 10:39 nvme1n1.blktrace.15
-rw-r--r--  1 root     root        5070264 Jun  2 10:39 nvme1n1.blktrace.16
-rw-r--r--  1 root     root        5075040 Jun  2 10:39 nvme1n1.blktrace.17
-rw-r--r--  1 root     root        5062104 Jun  2 10:39 nvme1n1.blktrace.18
-rw-r--r--  1 root     root        5586936 Jun  2 10:39 nvme1n1.blktrace.19
-rw-r--r--  1 root     root        3718848 Jun  2 10:39 nvme1n1.blktrace.2

```

## Step3
Now, we can use the **blkparse** to regenerate human-readable output form the output we get via **blktrace** before.

We need to indicate source files, you can just use the device name without **.blktrace.xx**, for example,
**nvmen1**, it will search all files which match the pattern **nvmen1.blktrace.xx** and put together to analyze.
Then, the **-f** option used to foramt the output data, you can find more about it via **man blkparse**

```
OUTPUT DESCRIPTION AND FORMATTING
       The output from blkparse can be tailored for specific use -- in particular, to ease parsing of output, and/or limit output fields to those the user wants to see. The data for fields which can be output include:

       a   Action, a (small) string (1 or 2 characters) -- see table below for more details

       c   CPU id

       C   Command

       d   RWBS field, a (small) string (1-3 characters)  -- see section below for more details

       D   7-character string containing the major and minor numbers of the event's device (separated by a comma).

       e   Error value

       m   Minor number of event's device.

       M   Major number of event's device.

       n   Number of blocks

       N   Number of bytes

       p   Process ID

       P   Display packet data -- series of hexadecimal values

       s   Sequence numbers

       S   Sector number

       t   Time stamp (nanoseconds)

       T   Time stamp (seconds)

       u   Elapsed value in microseconds (-t command line option)

       U   Payload unsigned integer
```

For our observation, we use **%5T.%9t, %p, %C, %a, %S\n** to format our result containing timestamp, command, process ID, action and sequence number.

Since the data I/O contains many action, such as complete, queued, inserted..ect. we can use option **-a** to filter actions, you can find more info via **man blktrace**.
In this case, we use the **write** to filter the actions.

In the end, use the **-o** options to indicate the output file name.

```
barrier: barrier attribute
complete: completed by driver
fs: requests
issue: issued to driver
pc: packet command events
queue: queue operations
read: read traces
requeue: requeue operations
sync: synchronous attribute
write: write traces
notify: trace messages
drv_data: additional driver specific trace
```

The command will look like below and it will output the result to file output.txt.

```
blkparse nvme1n1 -f "%5T.%9t, %p, %C, %a, %S\n"  -a write -o output.txt
```

open the file, the result looks like
```
    0.000000000, 22890, fio, Q, 1720960
    0.000001857, 22890, fio, G, 1720960
    0.000005803, 22890, fio, I, 1720960
    0.000009234, 22890, fio, D, 1720960
    0.000036821, 0, swapper/0, C, 1996928
    0.000067519, 22890, fio, Q, 1721088
    0.000068538, 22890, fio, G, 1721088
    0.000071531, 22890, fio, I, 1721088
    0.000073102, 22890, fio, D, 1721088
    0.000093464, 0, swapper/0, C, 1994624
    0.000123806, 0, swapper/0, C, 1785472
    0.000147436, 22892, fio, C, 1784576
    0.000159977, 22891, fio, C, 1997312
    0.000166653, 22891, fio, Q, 2006912
    0.000167632, 22891, fio, G, 2006912
    0.000169422, 22891, fio, I, 2006912
    0.000171178, 22891, fio, D, 2006912
    0.000188830, 22892, fio, Q, 1817728
    0.000189783, 22892, fio, G, 1817728
    0.000191405, 22892, fio, I, 1817728
    0.000192830, 22892, fio, D, 1817728
    0.000202367, 22891, fio, Q, 2007040
    0.000203160, 22891, fio, G, 2007040
    0.000205969, 22891, fio, I, 2007040
    0.000207524, 22891, fio, D, 2007040
    0.000227655, 22892, fio, Q, 1817856
    0.000228457, 22892, fio, G, 1817856
    0.000231936, 22892, fio, I, 1817856
....
```
Since the fio will fork to two process to handle the process, we use the **grep** to focus on one specific process (pid=22892).

```
grep "22892, fio" output.txt | more
```

Now, the result seems good, we can discover the sequence number (fifth column) is increasing.
One thing we need to care about is the row which action is "C", which means the completed, since we don't know how NVME handle those request and reply to upper layer. we only need to focus on other action. such as "Q (queued This notes intent to queue i/o at the given location.  No real requests exists yet.)" or "I (inserted A request is being sent to the i/o scheduler for addition to the internal queue and later service by the driver. The request is fully formed at this time)".

```
    0.000147436, 22892, fio, C, 1784576
    0.000188830, 22892, fio, Q, 1817728
    0.000189783, 22892, fio, G, 1817728
    0.000191405, 22892, fio, I, 1817728
    0.000192830, 22892, fio, D, 1817728
    0.000227655, 22892, fio, Q, 1817856
    0.000228457, 22892, fio, G, 1817856
    0.000231936, 22892, fio, I, 1817856
    0.000233530, 22892, fio, D, 1817856
    0.000360361, 22892, fio, Q, 1817984
    0.000361310, 22892, fio, G, 1817984
    0.000364163, 22892, fio, I, 1817984
    0.000366696, 22892, fio, D, 1817984
    0.000536731, 22892, fio, Q, 1818112
    0.000537758, 22892, fio, G, 1818112
    0.000539371, 22892, fio, I, 1818112
    0.000541407, 22892, fio, D, 1818112
    0.000670209, 22892, fio, Q, 1818240
    0.000671345, 22892, fio, G, 1818240
    0.000673383, 22892, fio, I, 1818240
    0.000676260, 22892, fio, D, 1818240
    0.001885543, 22892, fio, Q, 1818368
    0.001887444, 22892, fio, G, 1818368
    0.001891353, 22892, fio, I, 1818368
    0.001895917, 22892, fio, D, 1818368
    0.001934546, 22892, fio, Q, 1818496
    0.001935468, 22892, fio, G, 1818496
    0.001936891, 22892, fio, I, 1818496
    0.001938742, 22892, fio, D, 1818496
    0.001965818, 22892, fio, Q, 1818624

```

Now, we can do all above command again and change the **section** to rw for fio using the randon write pattern. The **blkparse** result will show the random sequence number.

# Summary
In this article, we try to use tools **blktrace** and **blkparse** to analysiz the block level I/O  for **fio** request.
We observe the filed **sequence number** to make sure thhat the fio can generate the **sequence** or **random** according to its config.


# Reference

- [539-blktrace-and-btt-example-to-debug-and-tune-disk-i-o-on-linux](http://fibrevillage.com/storage/539-blktrace-and-btt-example-to-debug-and-tune-disk-i-o-on-linux)
