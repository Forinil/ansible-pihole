# Pi-Hole

This Ansible role automates the installation of [Pi-Hole](https://pi-hole.net)
on your Ubuntu Jammy (22.04) or Focal (20.04) server.



## Usage

Please feel free to redefine any variable in the `pihole_default` dict, but you
must do so by creating a dict named `pihole` somewhere with higher priority,
such as group vars, host vars, or play vars.

**Note that you only need to specify the part(s) of this dict that you wish to edit.**
For example, to reset the minimally necessary variables, add the following somewhere as per above:
```
pihole:
  setupVars:
    webpassword: "abettermoreprivatepassword"
    dns_servers:
      - "127.0.0.1#5353"
```
To understand how the `pihole` dict is merged with `pihole_default`, see `./merge-defaults.yml`
(it is simple a recursive merge using the `combine()` filter, and works reliably in my experience).

But please note that unless you enable hash merging globally, this approach (with a single dict)
limits you to specifying your `pihole` dict in *one place only*. For example, you must not
set `pihole.ftlconfig.privacylevel` in host vars and `pihole.setupVars.dns_servers` in group vars
(in such a case, only values set in host vars would become part of the finally used `pihole`).
This is annoying, I agree, but globally enabling merging of dicts is considered poor practice as
far as I can understand.

+ https://github.com/ansible/ansible/issues/73089
+ https://reddit.com/r/ansible/comments/he1437/hash_merge_alternatives
+ https://old.reddit.com/r/ansible/comments/he1437/hash_merge_alternatives/


### Set the web-admin password

You must supply your own password to `pihole.setupVars.webpassword`.
Note that this role takes care of the [double-hashing that Pi-Hole requires](https://github.com/pi-hole/AdminLTE/blob/master/scripts/pi-hole/php/password.php#L58).

For reference, here is one way to double-hash a password provided from stdin:
```
echo -n somelongandsecretstringofyourown | sha256sum | awk '{printf "%s",$1 }' | sha256sum
```


## How to remotely access the Pi-Hole web-admin

Assuming you can SSH into the Pi-Hole host, just open up an SSH port forward
```
ssh -L 8800:localhost:80 alexandria
```

where we forward a local unprivileged port (e.g., 8800), to port 80 on the Pi-Hole host
(in this `alexandria` is configured in `~/.ssh/config`).

In a browser on your local machine, go to `http://localhost:8800/admin`. Voilá!

PS. The ssh forward command above can be improved by adding suitable flags.
Right now, it opens a terminal session, which is unnecessary.



## The FTL database can get quite large in a year

By default Pi-Hole saves `/etc/pihole/pihole-FTL.db` for a year, which for a
Raspberry Pi running off of an SD card can become a significant chunk of space.
On my DNS server it's currently >3GB, with `/var/log/` using an additional 2GB.

To restrict the FTL database, set `pihole.ftlconfig.maxdbdays` to a value less
than `365`.

+ https://reddit.com/r/pihole/comments/ioqvy3/can_i_delete_piholeftldb
+ https://reddit.com/r/pihole/comments/g4j0gh/deleted_by_user
+ https://docs.pi-hole.net/ftldns/configfile



## Links and notes

+ https://github.com/pi-hole/pi-hole
+ https://docs.pi-hole.net/ftldns/in-depth
+ https://blog.ktz.me/fully-automated-dns-and-dhcp-with-pihole-and-dnsmasq
+ https://www.billweber.io/2022/05/15/nova-lab-pihole-ansible
+ https://www.pudar.net/tag/ansible.html


### Unattended install

This role runs the Pi-Hole CLI install script in `unattended` mode.
Instead of configuring Pi-Hole via interactive prompts in the CLI, we create
`setupVars.conf` *before* running the unattended install script.

+ https://reddit.com/r/pihole/comments/13q1cqd/automated_install_headless_in_ansible
+ https://github.com/pi-hole/pi-hole/issues/5283
+ https://discourse.pi-hole.net/t/pi-hole-as-part-of-a-post-installation-script/3523


### Web server alongside Pi-Hole (Heimdall in my case)

+ [Using Pi-Hole docker container behind reverse proxy without it being the default host](https://discourse.pi-hole.net/t/using-pi-hole-docker-container-behind-reverse-proxy-without-it-being-the-default-host/39804)
+ [Set up a webpage on PiHole's lighttpd?](https://reddit.com/r/pihole/comments/5bvi29/set_up_a_webpage_on_piholes_lighttpd)
+ [Quick and dirty multi-server hosting alongside Pi-Hole using lighttpd](https://reddit.com/r/pihole/comments/6ouy8l/quick_and_dirty_multiserver_hosting_alongside)
+ [Run another site alongside Pi-hole?](https://reddit.com/r/pihole/comments/ex8vc9/run_another_site_alongside_pihole)
+ [How do I host a website alongside Pi-Hole?](https://discourse.pi-hole.net/t/how-do-i-host-a-website-alongside-pi-hole/4187)
+ [Pi-hole silently installs lighttpd and messes up the configuration of the existing web server #1785](https://github.com/pi-hole/pi-hole/issues/1785)
+ [Choice of web server (lighttpd vs manual configuration) #353](https://github.com/pi-hole/pi-hole/pull/353)
+ [lighttpd port change question](https://reddit.com/r/pihole/comments/aw5q1j/lighttpd_port_change_question)
+ [How to edit external.conf to override lightpd.conf default port](https://discourse.pi-hole.net/t/how-to-edit-external-conf-to-override-lightpd-conf-default-port/15712/1)


### VPN server (Wireguard or Tailscale) alongside Pi-Hole

+ [How to set up a Wireguard VPN server with Pi-Hole](https://marcocetica.com/posts/wireguard_pihole), Marco Cetica, 2022-08-09


### Ansible roles

+ https://gitlab.com/cbz-d-velop/public-ansible/ansible-role-labocbz-install-pihole (last commit 3 days ago)
+ https://github.com/dbrennand/home-ops/blob/dev/ansible/playbooks/pihole-playbook.yml (last commit 1 month ago)
+ https://github.com/zaszi/ansible-role-pihole (last commit 2 months ago)
+ https://github.com/drew1kun/ansible-role-pihole (last commit 2 years ago)
+ https://github.com/blalop/ansible-role-pihole (last commit 3 years ago)
+ https://github.com/fheinle/ansible-pihole (last commit 7 years ago)


### Ansible roles that install Pi-Hole from Docker image

+ https://github.com/shaderecker/ansible-pihole (last commit 2 days ago)
+ https://github.com/elgeeko1/elgeeko1-pihole-ansible (last commit 5 months ago)
+ https://github.com/timrwwatson/Ansible-Pi-Hole-Set-Up (last commit 2 years ago)
+ https://git.ducamps.eu/ansible-roles/ansible-pihole (last commit 2 years ago)
+ https://github.com/Freekers/automated-pihole (last commit 4 years ago)


### Suggestions

#### Sync Gravity across Pi-Hole instances

+ https://github.com/vmstan/gravity-sync
+ https://github.com/vmstan/gravity-sync/wiki

Sounds interesting, except I think I prefer each Pi-Hole to be independent.
Perhaps I will change my mind down the road...


#### Should this role configure DNSMASQ custom config?

Perhaps draw inspiration from [`drew1kun` role](https://github.com/drew1kun/ansible-role-pihole).
