Linux Server Configuration
==========================

This is an overview of some of the steps taken to configure a web server following Udacity's specifications as part of their [Full Stack Web Developer][] nanodegree program. Specifically, the setup of an [Apache][] server running a [Flask][] app hosted on an [Amazon Lightsail][] instance running [Ubuntu][].

Addresses
---------

Server URL:

SSH IP and port (for Udacity reviewer):

Software installed
------------------

Resources referenced
--------------------

When upgrading `apt` packages, there was a prompt about a new version of a grub config file. I decided to keep the existing version based on a comment here: https://forums.aws.amazon.com/thread.jspa?threadID=96207

Setup steps
-----------

* Create a Lightsail instance from <https://lightsail.aws.amazon.com/> and create a static IP for it in the **Networking** tab of the web dashboard.
* Add a barebones `.vimrc` for convenience.
* SSH in and change the ssh public key to my own (`vim ~/.ssh/authorized_keys`)
* Update packages
    - `sudo apt-get update && sudo apt-get upgrade && sudo apt-get autoremove`
* Change the SSH port
    - `sudo vim /etc/ssh/sshd_config` and change the line "`Port 22`" (to "`Port 2200`" in this case)
    - `sudo service ssh restart` to apply the changes
    - In the AWS dashboard (Networking tab), edit the firewall rules to add the new port for TCP connections and (optionally) delete the default port 22 rule.
* After confirming that non-root login works using public/private key and allows the use of `sudo`, disable root login and password login.
    - `sudo vim /etc/ssh/sshd_config`
        - Change appropriate line to `PermitRootLogin no`
        - Change appropriate line to `PasswordAuthentication no` (this was already set)
    - `sudo service ssh restart` to apply the changes
* Configure firewall (for a simple Lightsail server like this one, this step could be considered redundant since the same settings must also be made in the AWS dashboard -- and that firewall is hit before ever reaching our server -- but it's always good to make sure our server secures itself anyway).
    - Verify that firewall is *not* active yet: `sudo ufw status`
    - `sudo ufw default deny incoming && sudo ufw default allow outgoing`
    - `sudo ufw allow 2200/tcp` (or whatever port we set up for SSH)
    - `sudo ufw allow 80/tcp`
    - `sudo ufw allow 123/tcp` (for NTP; also add this in the AWS dashboard)
    - `sudo ufw enable`
* Create the "grader" user and give them `sudo` access
    - `sudo adduser grader`
    - `sudo EDITOR=vim visudo -f /etc/sudoers.d/90-cloud-init-users` (or copy the file and change the copy)
    - Create `/home/grader/.ssh/` and `/home/grader/.ssh/authorized_keys`, making each of them has "grader" as the owner and their permissions appropriately restricted.
    - Create a key pair locally using `ssh-keygen` and add the public key to `/home/grader/.ssh/authorized_keys`


[Amazon Lightsail]: https://aws.amazon.com/lightsail/
[Apache]: https://httpd.apache.org/
[Flask]: http://flask.pocoo.org/
[Full Stack Web Developer]: https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004
[Ubuntu]: https://www.ubuntu.com/
