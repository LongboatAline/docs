`no space left on device`

If your Edison has been running smoothly for a while, chances are that the filesystem is filling up.
* slowly: due to normal operation, and even maintenance,
* quickly: if something is off, and loggin processes are trying to tell you about it, but noone takes note of the logs...

Fortunately, there are some quick actions you can take to reclaim space from the most likely culprits 
before debugging the problem that caused your filesystem to fill up.
1. **Remove unused software archives**
   ```
   sudo apt-get clean
   ``` 
  * Reason: the packages have already been installed, and left on your filesystem in case you want to revert out of an upgrade. 
   If your edison has been running stable after the last upgrade, chances are you'll never need these files again. 
   (And if you do, the package-manager will download them again)
  * Example: The following example freed 240MB. Not enough for unattended use, but unless something is churning space rapidly,
    the system should be able to proceed while you're continuing down these instructions.
   ```
   root@antlia:/# du -sh /var/cache/*
   242M	/var/cache/apt
   ...
   root@antlia:/# apt-get clean
   root@antlia:/# du -sh /var/cache/*
   40K	/var/cache/apt
   ```
1. watch the free space now with the command `df /`. 
   Does the free space remain free, or is something eating at it with a voracious appetite? 
   Time to proceed to the second culprit and take a look at logfiles.
   * tl;dr: Logfiles reside in the directory `/var/log`, just as the package manager cache did. 
   This parent directory `/var` is named aptly, and reserved for files that have a tendency for fast, even exponential growth. 
   What does `du -sh /var/log` tell you? If there's more than a b
   
   
There are a few things you'll want to know:
1. deleting a file that is in use by a running program will not free the disk space it is occupying. 
   It will, however, deprive you of any _normal_ means to access the file and fix the problem.
   *  Reason: That running process usually has an _open_ `file handle` on the file, 
      and only after the file is _closed_, the space gets freed.
   *  Solution: Do not delete the file, instead, nuke its contents:
      ```
      :> <filename>
      ```
      (do not type the wedges around the filename, they have a special meaning! 
      DO type the `>` after the colon, though)
   * Explanation (tl;dr): `:` is the NOP of the shell, so the command will redirect "nothing" 
     into the file, effectively zeroing its contents.
   * Solution: if you already removed the file (aka your access to the file): 
     Find the process that is writing it, and restart it.
 2. To find all files that are currently growing, you can use `find`. Create a new file, wait a moment, and let find search for any file that has been written to more recently than that timestap file:
    ```
    :> timestamp
    find / -newer timestamp -ls
    ```
   
# WIP: Some notes to incorporate later or elsewhere
```
root@antlia:/# du -sh /var/cache/*
242M	/var/cache/apt
...
root@antlia:/# apt-get clean
root@antlia:/# du -sh /var/cache/*
40K	/var/cache/apt
...
root@antlia:/# df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/root        1.4G  282M  1.1G  22% /
```
* syslog
```
Feb 18 01:11:07 antlia CRON[7541]: (CRON) info (No MTA installed, discarding output)
```
If a cron job is producing output, that output gets sent by mail. If then the mailsystem is (intentionally, in our case) not configure to send that mail anywhere, it will log an error message per mail unsent.
Fix: either redirect output from cron jobs to a specific logfile or discard it alltogether.


nitpicking and gritty details:
* sloppy startup script
```
/etc/network/if-pre-up.d/wpasupplicant: cannot create /dev/stderr: No such device or address
```
Reason:
the startupscript tries to create /dev/stderr (which is a socket). 
([Here's the tl;dr](https://lists.freedesktop.org/archives/systemd-devel/2015-April/031245.html) on the reason why that has become a problem)
Instead of redirecting to /dev/stderr, it should redirect the filedescriptor instead:
replace  `> /dev/stderr` with `1>&2`

* auth.log
```
Feb 18 14:59:02 antlia CRON[1135]: pam_unix(cron:session): session closed for user edison
Feb 18 14:59:05 antlia CRON[1117]: pam_unix(cron:session): session closed for user root
```
