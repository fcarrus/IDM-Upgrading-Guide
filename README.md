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

You need to start from the *CA-Renewal Master* ([why?](https://access.redhat.com/solutions/4173861)).

```
# kinit admin

# ipa config-show | grep -i renewal
```

Login to the server and run tmux. Have one tmux-window (I prefer to use split-panes) for each of these:
* tail -f /var/log/dirsrv/*/errors
* top -d1 
* tail -f /var/log/ipaupgrade.log
* a free shell where you can run commands

**Note**: Don't avoid using tmux. It can really save your work if you get disconnected for any reason.

### Let's go

After stop all IDM services, other replicas should take care of the ipa-clients. The SRV records in the *_ldap._tcp.example.com* domain are there for this reason. To check them:

```
# host -t SRV _ldap._tcp.example.com
```

For each of the listed servers, login as root and take note of how many ldap connections are being served:

```
# ss -ntp | grep -e :389 -e :636 -c
```

It might be useful later when considering tuning.

Shutting down services is going to disrupt some system's login. It always happens, especially if the firewall has the nasty habit of *forgetting* the idle connections without informing the peers (i.e. no *TCP Reset*). This can result in the ipa-client losing time trying to use a non-existent connection and timing out.

Stop the IDM Services:

```
# ipactl stop
```

Now it's a good time to take a snapshot. Please note that even if reverting to a snapshot works, the database may go back to a state where it needs to be re-initialized. You will know if it happens because the `errors` log file will print lines like:

```
[19/May/2020:16:44:01.542938382 +0200] - ERR - agmt="cn=meToreplica97.example.com" (replica97:389) - clcache_load_buffer - Can't locate CSN 5ec29378009800070000 in the changelog (DB rc=-30988). If replication stops, the consumer may need to be reinitialized.
[19/May/2020:16:44:01.553776061 +0200] - ERR - NSMMReplicationPlugin - send_updates - agmt="cn=meToreplica97.example.com" (replica97:389): Missing data encountered. If the error persists the replica must be reinitialized.
```

To [reinitialize](https://access.redhat.com/solutions/452303) a replica from another working server:

```
[root@broken-replica]# ipa-replica-manage re-initialize --from good-replica.example.com
```

Set the new RHEL Release.  If you're just starting, don't point to the next one; first get to the latest packages of your current release:

```
# cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.4 (Maipo)
```

Set the release and clean the yum cache:

```
# subscription-manager release --set=7.4 && yum clean all && rm -rf /var/cache/yum
```


Update the sssd packages **only** ([why?](https://access.redhat.com/solutions/3417551)):

```
# yum -y update sssd\*
```

Restart SSSD cleaning its cache:
```
# systemctl stop sssd && rm -vf /var/lib/sss/db/* && systemctl  start sssd
```

Update the whole system:

```
# yum -y update
```

The Cleanup phase of the IDM packages contain post-update rpm scripts that run `ipa-server-upgrade`. You will see some activity on the `errors` log file. While yum is in this phase, keep an eye on log file and `ipaupgrade.log` file. 

The dirsrv's shutdown phase should happen several times, and they should never hang for more than 30sec - 1minute. A shutdown and restart of dirsrv usually takes way less than that.

If it keeps being hung for several minutes, it is definitely stuck. I don't know why. I tried `strace`'ing the stuck thread but it seemed POLL'ing without end, like a deadlock. In this case, try:

```
# top -d1 -H -p $(pidof ns-slapd)
```

take note of the uppermost thread, it should be running 100% or something like it.  Try `kill -9`ing that PID. The `errors` log should continue its start-stop-start until it has completed upgrading the schema.

When yum update is finished, keep looking at the `errors` log file. It will tell you the schema-compat-plugin tree scan will start. If then it says `waiting for SSSD to become online...` it means SSSD did not start correctly.  

If you run `id someuser@ad.example.com` now, it may say the user does not exist.  Restart the service with a cleanup:

```
# systemctl stop sssd && rm -vf /var/lib/sss/db/* && systemctl  start sssd
```

Wait a few seconds then try again resolving users with `id`. Also try to resolve SUDO rules:

```
# sudo -ll -U someuser@ad.example.com
```

If SSSD does not cooperate, try again restarting it with cache cleanup. It might take a few attempts.

At this point, all is OK but you need to reboot the server to use the new kernel. In any case, it's ok to reboot now.

```
# shutdown -r now
```

When the server comes back up, try again the `id` command and restart SSSD if it does not work, like said above. It has to work or the IDM services won't start.

Now run `ipa-server-upgrade`:

```
# ipa-server-upgrade -v
```

As always, keep an eye on the `errors` log. It should behave like when `yum update` was running.

If dirsrv gets stuck while shutting down, follow the steps above until it completes the upgrade steps.

When the ipa-server-upgrade completes successully, test again `id` and `sudo -ll` for some users. It should work.

**Note**: In some cases, it is advised to disable the compat plugin while running the upgrades.  I did so, but it completely disabled one of the most customer-used features: the [sudo-for-local-user](https://access.redhat.com/solutions/2347541). I skipped disabling it without major problems. The most evident is the `waiting for SSSD to become online...` message like said above.

### Repeat for All Other Replicas

Repeat on other replicas until all of them are at the same RHEL version. Do only **one server at a time**. Do not keep them on mixed releases (i.e. 7.4 and 7.5) for long time. This is not supported except while upgrading.

When all servers are done, go back to the _CA-Renewal Master_ and repeat for the next RHEL release.

Have fun.