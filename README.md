# Pi-Hole

This Ansible role automates the installation of [PiHole](https://pi-hole.net/).


## Setting the web password

Set in `setupVars.conf` using the `WEBPASSWORD` field.
The password is stored in this file in some hashed form.
[But how exactly, and how can we generate our own hash?](https://github.com/drew-kun/ansible-pihole/blob/0315891ed63e406318c5e33bdcc0443f37e8b396/defaults/main.yml)
Pi-Hole's web password must be stored as a [double SHA256 hash](https://github.com/pi-hole/AdminLTE/blob/master/scripts/pi-hole/php/password.php#L58).

To hash a password supplied from stdin:
```
echo -n P@ssw0rd | sha256sum | awk '{printf "%s",$1 }' | sha256sum
```



## How to open the Pi-Hole webadmin UI remotely

Assuming you can ssh into the Pi-Hole host, just open up an SSH port forward

```
ssh -L 8800:localhost:80 alexandria
```

where we forward a local unprivileged port (e.g., 8800), to port 80 on the Pi-Hole host.

In a browser on your local machine, go to `http://localhost:8800/admin`. Voilá!

PS. The ssh forward command above can be improved by adding suitable flags.
Right now, it opens a terminal session, which is unnecessary.



## Gravity-Sync(?)

https://github.com/vmstan/gravity-sync/
https://github.com/vmstan/gravity-sync/wiki

Sounds interesting, except I think I prefer each Pi-Hole to be independent.
Except perhaps for the FM Pi-Hole's...



## The FTL database can get quite large in a year

`/etc/pihole/pihole-FTL.db` can grow quite large over time.
On FS1 LANs DNS server, it's currently >3GB (the entire container uses 8GB, and
2GB of that is `/var/log/`).

+ https://reddit.com/r/pihole/comments/ioqvy3/can_i_delete_piholeftldb/
+ https://reddit.com/r/pihole/comments/g4j0gh/deleted_by_user/
+ https://docs.pi-hole.net/ftldns/configfile/


## Links

+ https://discourse.pi-hole.net/t/using-pi-hole-docker-container-behind-reverse-proxy-without-it-being-the-default-host/39804
+ https://reddit.com/r/pihole/comments/5bvi29/set_up_a_webpage_on_piholes_lighttpd/
+ https://reddit.com/r/pihole/comments/6ouy8l/quick_and_dirty_multiserver_hosting_alongside/
+ https://reddit.com/r/pihole/comments/ex8vc9/run_another_site_alongside_pihole/
+ https://discourse.pi-hole.net/t/how-do-i-host-a-website-alongside-pi-hole/4187
+ https://github.com/pi-hole/pi-hole/issues/1785
+ https://github.com/pi-hole/pi-hole/pull/353
+ https://reddit.com/r/pihole/comments/aw5q1j/lighttpd_port_change_question/
+ https://discourse.pi-hole.net/t/how-to-edit-external-conf-to-override-lightpd-conf-default-port/15712/1
+ https://marcocetica.com/posts/wireguard_pihole/
