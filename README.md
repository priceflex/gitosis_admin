gitosis_admin
=============

Update (08-10-2008): For topics not covered here, I encourage everyone to read the gitosis README, bundled with the distribution. There is also an example.conf configuration file that illustrates more features than I have covered here.

I have been asked more and more these days, “How do I host a Git repository?” Usually it is assumed that some access control beyond simply read-only is involved (some users have commit rights). With access control comes issues of security, and that’s a whole other bag of cats. This post is about presenting an answer to this question, without the fuss.

The rest of this article will be a tutorial showing you how to host and manage Git repositories with access control, easily and safely. I use an up and coming tool called gitosis that my friend Tv wrote to help make hosting git repos easier and safer. It manages multiple repositories under one user account, using SSH keys to identify users. However, users do *not* need shell accounts on the server, instead they will talk to one shared account that does not allow arbitrary commands. Git itself is used to setup gitosis and manage the Git repos, which pleases the recursion-seeking orthogonal CS-side of my brain.

Assumptions: I take my examples from a Ubuntu Linux server. While I haven’t tested other systems, I imagine different Linux distributions, FreeBSD, OS X, etc… would be similar. Gitosis is written in Python, so you should have a copy of Python installed as well.

Enough talk, let’s get down and dirty.

Install gitosis

gitosis is a tool for hosting git repositories (I’m repeating myself for those who skim :)

The first thing to do is grab a copy of gitosis and install it on your server:

1
2
cd ~/src
git clone git://eagain.net/gitosis.git
Notice that gitosis is extremely light-weight. The clone takes a mere couple seconds. Less is more, and I like that a lot.

Next, install it like so:

1
2
cd gitosis
python setup.py install
Don’t use –prefix unless you like self-inflicted pain. It is possible to install gitosis in a non-standard location, but it’s not nice. Read the Caveats section at the bottom and then come back here.

If you get this error:

1
-bash: python: command not found
or

1
2
3
4
Traceback (most recent call last):
  File "setup.py", line 2, in ?
    from setuptools import setup, find_packages
ImportError: No module named setuptools
You have to install Python setuptools. On Debian/Ubuntu systems, it’s just:

1
sudo apt-get install python-setuptools
For other systems, someone tell me or leave a comment, so I can update this section and improve this tutorial.

The next thing to do is to create a user that will own the repositories you want to manage. This user is usually called git, but any name will work, and you can have more than one per system if you really want to. The user does not need a password, but does need a valid shell (otherwise, SSH will refuse to work).

1
2
3
4
5
6
7
8
sudo adduser \
    --system \
    --shell /bin/sh \
    --gecos 'git version control' \
    --group \
    --disabled-password \
    --home /home/git \
    git
You may change the home path to suit your taste. A successful user creation will look similar to:

1
2
3
4
Adding system user `git'...
Adding new group `git' (211).
Adding new user `git' (211) with group `git'.
Creating home directory `/home/git'.
You will need a public SSH key to continue. If you don’t have one, you may generate one on your local computer:

1
ssh-keygen -t rsa
The public key will be in $HOME/.ssh/id_rsa.pub. Copy this file to your server (the one running gitosis).

Next we will run a command that will sprinkle some magic into the home directory of the git user and put your public SSH key into the list of authorized keys.

1
sudo -H -u git gitosis-init < /tmp/id_rsa.pub
id_rsa.pub above is your public SSH key that you copied to the server. I recommend you put it in /tmp so the git user won’t have permission problems when trying to read it. Success looks like:

1
2
Initialized empty Git repository in ./
Initialized empty Git repository in ./
(Yes, two times)

For good measure, let’s make sure the post-update hook is set executable. I’ve seen it where sometimes it doesn’t get set (problem with older setuptools):

1
sudo chmod 755 /home/git/repositories/gitosis-admin.git/hooks/post-update
Here some cool magic happens. Run this on your local machine:

1
2
git clone git@YOUR_SERVER_HOSTNAME:gitosis-admin.git
cd gitosis-admin
You will now have a gitosis.conf file and keydir/ directory:

1
2
3
4
~/dev/gitosis-admin (master) $ ls -l
total 8
-rw-r--r--   1 garry  garry  104 Nov 13 05:43 gitosis.conf
drwxr-xr-x   3 garry  garry  102 Nov 13 05:43 keydir/
This repository that you just cloned contains all the files (right now, only 2) needed to create repositories for your projects, add new users, and defined access rights. Edit the settings as you wish, commit, and push. Once pushed, gitosis will immediately make your changes take effect on the server. So we’re using Git to host the configuration file and keys that in turn define how our Git hosting behaves. That’s just plain cool.

From this point on, you don’t need to be on your server. All configuration takes place locally and you push the changes to your server when you’re ready for them to take effect.

Creating new repositories

This is where the fun starts. Let’s create a new repository to hold our project codenamed FreeMonkey.

Open up gitosis.conf and notice the default configuration:

1
2
3
4
5
[gitosis]                  

[group gitosis-admin]
writable = gitosis-admin
members = jdoe
Your “members” line will hold your key filename (without the .pub extension) that is in keydir/. In my example, it is “jdoe”, but for you it’ll probably be a combination of your username and hostname.

To create a new repo, we just authorize writing to it and push. To do so, add this to gitosis.conf:

1
2
3
[group myteam]
members = jdoe
writable = free_monkey
This defines a new group called “myteam”, which is an arbitrary string. “jdoe” is a member of myteam and will have write access to the “free_monkey” repo.

Save this addition to gitosis.conf, commit and push it:

1
2
git commit -a -m "Allow jdoe write access to free_monkey"
git push
Now the user “jdoe” has access to write to the repo named “free_monkey”, but we still haven’t created a repo yet. What we will do is create a new repo locally, and then push it:

1
2
3
4
5
6
7
8
mkdir free_monkey
cd free_monkey
git init
git remote add origin git@YOUR_SERVER_HOSTNAME:free_monkey.git

# do some work, git add and commit files

git push origin master:refs/heads/master
With the final push, you’re off to the races. The repository “free_monkey” has been created on the server (in /home/git/repositories) and you’re ready to start using it like any ol’ git repo.

Adding users

The next natural thing to do is to grant some lucky few commit access to the FreeMonkey project. This is a simple two step process.

First, gather their public SSH keys, which I’ll call “alice.pub” and “bob.pub”, and drop them into keydir/ of your local gitosis-admin repository. Second, edit gitosis.conf and add them to the “members” list.

1
2
3
4
cd gitosis-admin
cp ~/alice.pub keydir/
cp ~/bob.pub keydir/
git add keydir/alice.pub keydir/bob.pub
Note that the key filename must have a “.pub” extension.

gitosis.conf changes:

1
2
3
4
[group myteam]
- members = jdoe
+ members = jdoe alice bob
  writable = free_monkey
Commit and push:

1
2
git commit -a -m "Granted Alice and Bob commit rights to FreeMonkey"
git push
That’s it. Alice and Bob can now clone the free_monkey repository like so:

1
git clone git@YOUR_SERVER_HOSTNAME:free_monkey.git
Alice and Bob will also have commit rights.

Public access

If you are running a public project, you will have your users with commit rights, and then you’ll have everyone else. How do we give everyone else read-only access without fiddling w/ SSH keys?

We just use git-daemon. This is independent of gitosis and it comes with git itself.

1
sudo -u git git-daemon --base-path=/home/git/repositories/ --export-all
This will make all the repositories you manage with gitosis read-only for the public. Someone can then clone FreeMonkey like so:

1
git clone git://YOUR_SERVER_HOSTNAME/free_monkey.git
To export only some repositories and not others, you need to touch git-daemon-export-ok inside the root directory (e.g. /home/git/repositories/free_monkey.git) of each repo that you want public. Then remove “–export-all” from the git-daemon command above.

More tricks

gitosis.conf can be set to do some other neat tricks. Open example.conf in the gitosis source directory (where you originally cloned gitosis way at the top) to see a summary of all options. You can specify some repos to be read-only (opposite of writable), but yet not public. A group members list can include another group. And a few other tricks that I’ll leave it to the reader to discover.

Caveats

If /home/git/.gitosis.conf on your server never seems to get updated to match your local copy (they should match), even though you are making changes and pushing, it could be that your post-update hook isn’t executable. Older versions of setuptools can cause this. Be sure to fix that:

1
sudo chmod 755 /home/git/repositories/gitosis-admin.git/hooks/post-update
If your Python goodies are in a non-standard location, you must additionally edit post-update and put an “export PYTHONPATH=…” line at the top. Failure to do so will give you a Python stack trace the first time you try to push changes within gitosis-admin.

If you want to install gitosis in a non-standard location, I don’t recommend it. It’s an edge case that the author hasn’t run up against until I bugged him to help me get it working.

For the brave, you need to edit whatever file on your system controls the default PATH for a non-login, non-interactive shell. On Ubuntu this is /etc/environment. Add the path to gitosis-serve to the PATH line. Also insert a line for PYTHONPATH and set it to your non-standard Python site-packages directory. As an example, this is my /etc/environment:

1
2
PATH="/home/garry/sys/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/bin/X11:/usr/games"
PYTHONPATH=/home/garry/sys/lib/python2.4/site-packages
Be sure to logout and log back in after you make these changes.

Don’t use the gitosis-init line I have above for the standard install, instead use this slightly modified one:

1
sudo -H -u git env PATH=$PATH gitosis-init < /tmp/id_rsa.pub
Be sure to also set PYTHONPATH in your post-update hook as described above.

The *should* do it. I am purposefully terse with this non-standard setup as I think not many people will use it. HIt me up in #git on FreeNode if you need more info (my nick is up_the_irons).

Non-standard SSH port

If you run SSH on a non-standard port on your server, don’t use the syntax “git@myserver.com:1234:/foo/bar”, it won’t work. Putting the port in the URL doesn’t seem to make gitosis, or git, (not sure which) happy. Instead, put this in your ~/.ssh/config file:

1
2
Host myserver.com
  Port 1234
Conclusion

Well, I hope you enjoyed this tutorial and that it makes your Git hosting life easier. This is public revision 10. Please let me know if you find mistakes, omissions, an easier way than what I described, etc…

Git it on…

