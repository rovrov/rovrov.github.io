---
layout: post
category: tools
tags: [tools, rsync, ssh]
date: 2014-11-08 21:00:00 -0500
---
{% include JB/setup %}

How can I synchronize a directory on my Windows system to a remote Linux machine, and then update the remote copy efficiently when files in the directory are changed? Nowadays a common answer is to use Dropbox or the like - install it on both systems and let it do its magic. It works, but what are other options without using a third-party serivce?

One way is to set up a git repository on Linux, a working directory on Windows and then `git push` changes to the repo. A drawback is that a commit must be done before each synchronization, and every version of each file will be stored in the repository which is not always necessary.

## rsync

Another approach is to use the [rsync](http://en.wikipedia.org/wiki/Rsync) tool originally created for Unix-like OS but a couple of Windows ports exist such as [cwRsync](http://google.com/search?q=cwRsync+free) (proprietary) and [DeltaCopy](http://google.com/search?q=deltacopy+rsync) (presumably open source). Both are cygwin-based, although do not require a complete cygwin installation being bundled with the required dlls.

<!-- more -->

Here is the command line I ended up using to synchronize a directory to a remote `linuxhost` where `myuser` is an existing user on that system for the ssh connection:

{% highlight bash %}
rsync --stats --delete --chmod=Du=rwx,Dg=rx,Do=rx,Fu=rw,Fg=r,Fo=r --modify-window=5 --exclude=".git" -c -r /cygdrive/C/win/path/ myuser@linuxhost:/home/user/linux_path
{% endhighlight %}

`--stats` provides some stats about the transfer, which I find useful:

{% highlight bash %}
...
Total bytes sent: 3,573
Total bytes received: 130

sent 3,573 bytes  received 130 bytes  1,481.20 bytes/sec
total size is 341,524  speedup is 92.23
{% endhighlight %}

`--delete` is required to ensure files deleted from local directory are deleted remotely as well. Otherwise, a file synced once and then removed from the local directory will not be deleted remotely.

`--chmod` specifies permissions assigned to directories and files created during synchronization.

`--modify-window=5` is a workaround for differences in handling timestamps between Linux and Windows file systems.  Otherwise, rsync thinks all files have changed and [sends the entire contents](http://serverfault.com/questions/291818/linux-ntfs-to-ntfs-rsync-repeatedly-recopying-files). Stats above indicate this actually works.

`--exclude=".git"` excludes the `.git` sub-directory from synchronization. My local directory is a git repo and I only need the contents synced, without the metadata.

`-c` *skips* files from synchronization based on checksum instead of the default timestamp/size. I first thought that it should resolve the issue described in `modify-window`, but it didn't.

`-r` for recursive mode, of course.

## ssh authentication

The rsync command above works over an ssh connection, and thus requires to authenticate each time it is run which is not extremely convenient. In case of key-based authentication which I am using, it prompts for a passphrase

{% highlight bash %}
Enter passphrase for key '/cygdrive/c/Users/winuser/.ssh/id_rsa':
{% endhighlight %}

Now, this can be avoided by either setting an empty passphrase or using an [ssh-agent](http://en.wikipedia.org/wiki/Ssh-agent) (pageant for PuTTY). Empty passphrase adds risk of identity theft if the key file is exposed. Using ssh-agent is somewhat better, but this utility is not included in the rsync distributions I mentioned. My git distribution has `ssh-agent.exe` in it (msys-based), but for some reason it didnt't work with cwRsync when I tried. Maybe will find something later, typing the passphrase is not that big of a deal for now.