# IDM Upgrading Guide

This guide was written based on Red Hat Knowledge Base articles and on-the-field experience made of trials, errors and near disasters.

It assumes you're in possession of root credentials, IDM admin credentials, and the Directory Server's _cn=Directory Manager_ password, as some commands will ask you for those.

It also assumes you're dealing with a multi-replica environment, from 2 servers onward.

Hope it helps. It can be a messy task when done in a production environment, so be patient. People will be mad at you.

## General Preparation

* Disable/stop anything that automatically changes configuration files (i.e. Puppet, Ansible Roles etc.)
* If you're using Satellite, enable and synchronize all releases of rhel-7-server-rpms from your current one (included) to the latest (7.8 at the time of this writing).
* Install tmux or screen (tmux is better) and learn how to use it. You'll be glad you did it.
* Ensure you don't have near-full filesystems in general on all replicas. It can be dangerous if /var/lib/dirsrv fills up, it could break your ldap database.
* Subscribe your replicas to RHSM/Satellite and attach a RHEL subscription if you haven't already done so.

## The Upgrade Sequence

You cannot just jump to the latest RHEL release (i.e. 7.4 -> 7.8). You have to proceed release by release, i.e:

* 7.4 -> 7.4.latest
* 7.4.latest -> 7.5.latest
* ...
* 7.7.latest -> 7.8.latest
* 7.8.latest -> 7.9.latest

You need to start from the *CA-Renewal Master* ([why?](https://access.redhat.com/solutions/4173861)).

To retrieve the *CA-Renewal Master Server*:

```
# kinit admin

# ipa config-show | grep -i renewal
```

Login to the server and run tmux. Open 4 tmux-windows (I prefer to use split-panes) for each of these. 

* tail -f /var/log/dirsrv/*/errors
* top -d1 
* tail -f /var/log/ipaupgrade.log
* a free shell where you can run commands

**If you don't want to use tmux and its windows, at least use it on the shell you're using for the update commands, and use other ssh logins for the other commands.**

### Briefely on tmux

* To create a new tmux session, just run `tmux`. You'll see a green bar at the bottom.
* To see if there are tmux sessions already open, run `tmux ls`.
* To get into a previous session, run `tmux a` (attach).
* To see if you are in a tmux session, `echo $TMUX`. If something is printed, you are in. Also, the green bar at the bottom is an hint.
* While you are in a tmux session, use `Ctrl+b` and then press `c` to create a new tmux-window.
* While you are in a tmux session, use `Ctrl+b` and then press `n` to cycle to the next window. Use `p` instead to go cycle backwards.
* To exit a tmux session leaving the session open, `Ctrl+b` and then `d` (detach).
* To close a tmux session, type `exit` on each one of the tmux windows.

**Note**: **Don't avoid using tmux**. It can really save your work if you get disconnected for any reason.

### Let's go

First, retrieve all the servers in the IDM cluster. They should be defined in a DNS entry as SRV records. To see all the replicas available:

```
# host -t SRV _ldap._tcp.example.com
```

(_Replace example.com with your domain_)

If you stop all IDM services on a server, the other replicas should take care of the ipa-clients. The SRV records in the *_ldap._tcp.example.com* domain are there for this reason. 

Starting with the CA-Renewal Master Server (see above), for each of the listed servers, login as root and open the tmux windows (see above), and run this command to see how many ldap connections are being served:

```
# ss -ntp | grep -e :389 -e :636 -c
```

Take note of the number, it might be useful later if you are considering tuning.

Stop the IDM Services on the current server:

```
# ipactl stop
```

Now it's a good time to take a snapshot. Please note that even if reverting to a snapshot works, the database may go back to a state where it needs to be re-initialized. You will know if it happens because the `errors` log file will print lines like:

```
[19/May/2020:16:44:01.542938382 +0200] - ERR - agmt="cn=meToreplica97.example.com" (replica97:389) - clcache_load_buffer - Can't locate CSN 5ec29378009800070000 in the changelog (DB rc=-30988). If replication stops, the consumer may need to be reinitialized.
[19/May/2020:16:44:01.553776061 +0200] - ERR - NSMMReplicationPlugin - send_updates - agmt="cn=meToreplica97.example.com" (replica97:389): Missing data encountered. If the error persists the replica must be reinitialized.
```

If you need to [reinitialize](https://access.redhat.com/solutions/452303) a replica from another working server you can run `ipa-replica-manage re-initialize --from good-replica.example.com` on the broken replica.

Set the RHEL Release to the next version you're currently on. To get the current release:

```
# cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.8 (Maipo)
```

Set the next release and clean the yum cache:

```
# subscription-manager release --set=7.9 && yum clean all && rm -rf /var/cache/yum
```


Update the sssd packages **only** ([why?](https://access.redhat.com/solutions/3417551)):

```
# yum -y update sssd\*
```

Restart SSSD with cache cleanup:
```
# systemctl stop sssd && rm -vf /var/lib/sss/db/* && systemctl start sssd
```

Now update the whole system (all RHEL and IDM related packages):

```
# yum -y update
```

### Some troubleshooting 

The Cleanup phase of the IDM packages contain post-update rpm scripts that run `ipa-server-upgrade`. You will see some activity on the `errors` log file. While yum is in this phase, keep an eye on log file and `ipaupgrade.log` file. 

The dirsrv's shutdown phase should happen several times, and they should never hang for more than 30sec - 1minute. A shutdown and restart of dirsrv usually takes way less than that.

If it keeps being hung for several minutes, it is definitely stuck. I don't know why. I tried `strace`'ing the stuck thread but it seemed POLL'ing without end, like a deadlock. In this case, try:

```
# top -d1 -H -p $(pidof ns-slapd)
```

take note of the uppermost thread, it should be running 100% or something like it.  Try `kill -9`ing that PID. The `errors` log should continue its start-stop-start until it has completed upgrading the schema.

When yum update is finished, keep looking at the `errors` log file. It will tell you the schema-compat-plugin tree scan will start. If it hangs saying `waiting for SSSD to become online...`, it means SSSD did not start correctly.   

If you run `id someuser@ad.example.com` now, it may say the user does not exist.  Restart the service with a cleanup:

```
# systemctl stop sssd && rm -vf /var/lib/sss/db/* && systemctl  start sssd
```

Wait a few seconds, then try again resolving users with `id`. It will work eventually.

Now try to resolve SUDO rules:

```
# sudo -ll -U someuser@ad.example.com
```

If still `id` and `sudo -ll` do not work, try again restarting SSSD with cache cleanup. It might take a few attempts.

At this point, all is OK but you need to reboot the server to use the new kernel. It's ok to reboot now.

```
# shutdown -r now
```

When the server comes back up, try again the `id` command and restart SSSD if it does not work, like said above. This has to work, otherwise the IDM services won't start correctly.

Now run `ipa-server-upgrade` one last time:

```
# ipa-server-upgrade -v
```

As always, keep an eye on the `errors` log. It should do the same like when `yum update` was running.

If dirsrv gets stuck while shutting down, follow the steps above until it completes the upgrade steps.

When the ipa-server-upgrade completes successully, test again `id` and `sudo -ll` for some users. It should work.

**Note**: In some cases, it is advised to disable the _compat plugin_ while running the upgrades.  I did so, but it completely disabled one of the most customer-used features: the [sudo-for-local-user](https://access.redhat.com/solutions/2347541). I skipped disabling it without major problems. The most evident is the `waiting for SSSD to become online...` message like said above.

### Repeat for All Other Replicas

Repeat on other replicas until all of them are at the same RHEL version. Do only **one server at a time**. Do not keep them on mixed releases (i.e. 7.4 and 7.5) for long time. This is not supported except while upgrading.

When all servers are done, repeat for the next RHEL Release starting from the _CA-Renewal Master_.

Have fun.