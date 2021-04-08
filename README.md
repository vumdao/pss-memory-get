<p align="center">
  <a href="https://dev.to/vumdao">
    <img alt="SES Email Tracking" src="https://github.com/vumdao/pss-memory-get/blob/master/cover.jpg?raw=true" width="700" />
  </a>
</p>
<h1 align="center">
  <div><b>Memory Consumption in Linux</b></div>
</h1>

## This article explains a reasonable method to measure memory consumption of a process on Linux. Linux is equipped with virtual memory management and, therefore, measuring the memory consumption of a single process is not as simple as most users think. This article explains what information you can get from each indicator related to memory consumption.

## The Linux tools most commonly used are the VSZ (Virtual Memory Size) and RSS (Resident Set Size), and the new one is Proportional set size (PSS)

## What’s In This Document 
- [Technical Terms](#-Technical-Terms)
- [VSZ (Virtual Memory Size) and Demand Paging](#-VSZ-(Virtual-Memory-Size)-and-Demand-Paging)
- [RSS (Resident Set Size) and Shared Libraries](#-RSS-(Resident-Set-Size)-and-Shared-Libraries)
- [PSS (Proportional Set Size)](#-PSS-(Proportional-Set-Size))
- [Python script to get PSS (Proportional Set Size)](#-Python-script-to-get-PSS-(Proportional-Set-Size))

---

### 🚀 **Technical Terms**
**1. page**
- This is a block of memory that is used in memory management on Linux. One page is 4096 bytes in typical Linux systems.

**2. physical memory**
- This is the actual memory, typically the RAM, that is on the computer.

**3. virtual memory**
- This is a memory space given to a process that lets the process think it has its own continuous memory that is isolated from other processes regardless of the actual memory amount on the computer or the situation of other processes memory consumption. A virtual memory page can be mapped to a physical memory page, and hence, processes only need to think about the virtual memory.

### 🚀 **VSZ (Virtual Memory Size) and Demand Paging**
- Considering the VSZ (virtual memory size) to measure memory consumption of a process does not make much sense. This is due to the feature called demand paging, which suppresses unnecessary memory consumption.

- For example, a text editor named emacs has functions that can handle XML files. These function, however, are not used all the time. Loading these functions on to the physical memory is not necessary when the user just wants to edit a plain text. The demand paging feature does not load pages unless they are used by the process.

- This is how it works. First, when the program starts, Linux gives a virtual memory space to the process but does not actually load pages that have functions on to the physical memory. When the program actually calls a function in the virtual memory, the MMU in the CPU tells Linux that the page is not loaded. Then Linux pauses the process, loads the page on to the physical memory, maps the page to the virtual memory of the process, then lets the process run again from where it got paused. The process, therefore, does not need to know that it got paused, and just simply assume the function was loaded on the virtual memory and use it.

- VSZ (virtual memory size) describes the entire virtual memory size of the process regardless of pages being loaded on the actual memory or not. This is, therefore, not a realistic indicator to measure memory consumption since it includes pages that are not actually consumed.

### 🚀 **RSS (Resident Set Size) and Shared Libraries**
- RSS (Resident Set Size) describes the total amount of the pages for a process that are actually loaded on the physical memory. This may sound like the real amount of memory being consumed by the process, and is better than VSZ (virtual memory size), but it is not that simple due to the feature called shared libraries or dynamic linking libraries.

- A library is a module that can be used by programs to handle a certain feature. For example, libpng.so takes care of compressing and decompressing PNG image files, and libxml2.so takes care of handling XML files. Instead of making each programmer write these functions, they can use libraries developed by others and achieve the result they want.

- A shared object is a library that can be shared by multiple programs or processes. For example, let's say there are two processes running at the same time that want to use XML handling functions that are in the shared library libxml2.so. Instead of loading the pages that have the exact same functions multiple times, Linux loads it once on to the physical memory and maps it to both processes virtual memory. Both processes do not need to care if they are sharing the functions with somebody else because they can access the functions and use them inside their own virtual memory. Due to this feature, Linux suppresses unnecessary duplication of memory pages.

- Now, let us go back to the same example above. Emacs, a text editor, has functions that can handle XML files. This is taken care of by the shared library libxml2.so. This time, the user that is running emacs is actually working with XML files and emacs is using the functions in libxml2.so. Meanwhile, there are two more process running in the background that are using libxml2.so too. Since libxml2.so is a shared library, Linux only loads it once on the physical memory and maps it to all three processes virtual memory.

- When you see the RSS (Resident Set Size) of emacs, it will include the pages of libxml2.so. This is not wrong because emacs is actually using it. But what about the other two processes? It is not just emacs that is using those functions. If you sum the RSS (Resident Set Size) of all three processes, libxml2.so will be counted three times even though it is only loaded on the physical memory once.

- RSS (Resident Set Size), therefore, is an indicator that will show the memory consumption when the process is running by it self without sharing anything with other processes. For practical situations where libraries are being shared, RSS (Resident Set Size) will over estimate the amount of memory being consumed by the process. Using to measure memory consumption of a process is not wrong but you may want to keep in mind of this behaviour.

### 🚀 **PSS (Proportional Set Size)**
- PSS (Proportional Set Size) is a relatively new indicator that can be used to measure memory consumption of a single process. It is not be available on all Linux systems yet but if it is available, it may come in handy. The concept is to split the memory amount of shared pages evenly among the processes that are using them.

- This is how PSS (Proportional Set Size) calculates memory consumption: If there are N processes that are using a shared library, each process is consuming one N-th of the shared libraries pages.

- For the example above, emacs and two other processes were sharing the pages of libxml2.so. Since there are three processes, PSS will consider each process is consuming one third of libxml2.so's pages.

- I consider PSS (Proportional Set Size) as a more realistic indicator compared to RSS (Resident Set Size). It works well especially when you want to consider the memory consumption of an entire system all together, and not each process individually. For example, when you are developing a system that has multiple processes and daemons and you want to estimate how much memory you should install on the device, PSS (Proportional Set Size) works better than RSS (Resident Set Size).

### 🚀 **Python script to get PSS (Proportional Set Size**
https://github.com/vumdao/pss-memory-get/pss.py

```
#! /usr/bin/env python3
# coding: utf-8
##-----------------------------------------------------------------------------
## pss.py --- Print the PSS (Proportional Set Size) of accessable processes
##-----------------------------------------------------------------------------
import os, sys, re, pwd
from functools import cmp_to_key as cmp


##-----------------------------------------------------------------------------
def pss_main():
    '''
    Print the user name, pid, pss, and the command line for all accessable
    processes in pss descending order.
    '''
    # Get the user name, pid, pss, and the command line information for all
    # processes that are accessable. Ignore processes where the permission is
    # denied.
    ls = []   # [(user, pid, pss, cmd)]
    for pid in filter(lambda x: x.isdigit(), os.listdir('/proc')):
        try:
            ls.append((owner_of_process(pid), pid, pss_of_process(pid), cmdline_of_process(pid)))
        except IOError:
            pass

    # Calculate the max length of the user name, pid, and pss in order to
    # print them in aligned columns.
    userlen = 0
    pidlen = 0
    psslen = 0
    for (user, pid, pss, cmd) in ls:
        userlen = max(userlen, len(user))
        pidlen = max(pidlen, len(pid))
        psslen = max(psslen, len(str(pss)))
    
    # Get the width of the terminal.
    with os.popen('tput cols') as fp:
        term_width = int(fp.read().strip())
    
    # Print the information. Ignore kernel modules since they allocate memory
    # from the kernel land, not the user land, and PSS is the memory
    # consumption of processes in user land.
    fmt = '%%-%ds  %%%ds  %%%ds  %%s' % (userlen, pidlen, psslen)
    print(fmt % ('USER', 'PID', 'PSS', 'COMMAND'))
    for (user, pid, pss, cmd) in sorted(ls, key=cmp(lambda x, y: (y[2] - x[2]))):
        if cmd != '':
            print((fmt % (user, pid, pss, cmd))[:term_width - 1])


##-----------------------------------------------------------------------------
def pss_of_process(pid):
    '''
    Return the PSS of the process specified by pid in KiB (1024 bytes unit)
    
    @param pid  process ID
    @return     PSS value
    '''
    with open('/proc/%s/smaps' % pid) as fp:
        return sum([int(x) for x in re.findall('^Pss:\s+(\d+)', fp.read(), re.M)])


##-----------------------------------------------------------------------------
def cmdline_of_process(pid):
    '''
    Return the command line of the process specified by pid.

    @param pid  process ID
    @return     command line
    '''
    with open('/proc/%s/cmdline' % pid) as fp:
        return fp.read().replace('\0', ' ').strip()


##-----------------------------------------------------------------------------
def owner_of_process(pid):
    '''
    Return the owner of the process specified by pid.

    @param pid  process ID
    @return     owner
    '''
    try:
        owner_pid = pwd.getpwuid(os.stat('/proc/%s' % pid).st_uid).pw_name
    except Exception:
        return 'docker'
    return owner_pid


##-----------------------------------------------------------------------------
if __name__ == '__main__':
    pss_main()
```

- How to run
`sudo python pss.py`

- If the server/instance running `docker` then it would show userID `999` [ref](https://stackoverflow.com/questions/55241474/why-docker-compose-creates-directories-files-with-usergroup-999999)

[Refs](https://web.archive.org/web/20120520221529/http://emilics.com/blog/article/mconsumption.html)
---

<h3 align="center">
  <a href="https://dev.to/vumdao">:stars: Blog</a>
  <span> · </span>
  <a href="https://github.com/vumdao/">Github</a>
  <span> · </span>
  <a href="https://vumdao.hashnode.dev/">Web</a>
  <span> · </span>
  <a href="https://www.linkedin.com/in/vu-dao-9280ab43/">Linkedin</a>
  <span> · </span>
  <a href="https://www.linkedin.com/groups/12488649/">Group</a>
  <span> · </span>
  <a href="https://www.facebook.com/CloudOpz-104917804863956">Page</a>
  <span> · </span>
  <a href="https://twitter.com/VuDao81124667">Twitter :stars:</a>
</h3>
