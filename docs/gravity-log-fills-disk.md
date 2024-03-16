# Issue: gravity update cron job log file fills disk

> Also see https://codeberg.org/ansible/pihole/issues/2

After the gravity update cron job triggers (Sunday morning by default) the generated
log file `/var/log/pihole_updateGravity.log` quickly fills the entire disk.

```
# grep -i updategravity /etc/cron.d/pihole
37 4   * * 7   root    PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole updateGravity >/var/log/pihole/pihole_updateGravity.log || cat /var/log/pihole/pihole_updateGravity.log
```

Immediately starts printing lines containing only the letter `y` at a furious rate.
Let's try to catch the output:
```
root@fmj-dns:~
# pihole updateGravity | tee updategravity.log
```
Yes, the output consists of only lines of `y`, nothing else. Most curious.

So something is wrong with the `pihole updateGravity` [command](https://docs.pi-hole.net/core/pihole-command/#gravity). Let's run the script itself that `updateGravity` calls in [verbose mode](https://github.com/pi-hole/pi-hole/issues/4422#issuecomment-989035923):
```
# bash -x /opt/pihole/gravity.sh > >(tee -a gravitysh.stdout) 2> >(tee -a gravitysh.err >&2)
# cat gravitysh.err
+ export LC_ALL=C
+ LC_ALL=C
+ coltable=/opt/pihole/COL_TABLE
+ source /opt/pihole/COL_TABLE
++ [[ -t 1 ]]
++ [[ -n '' ]]
++ COL_BOLD=
++ COL_ULINE=
++ COL_NC=
++ COL_GRAY=
++ COL_RED=
++ COL_GREEN=
++ COL_YELLOW=
++ COL_BLUE=
++ COL_PURPLE=
++ COL_CYAN=
++ COL_WHITE=
++ COL_BLACK=
++ COL_LIGHT_BLUE=
++ COL_LIGHT_GREEN=
++ COL_LIGHT_CYAN=
++ COL_LIGHT_RED=
++ COL_URG_RED=
++ COL_LIGHT_PURPLE=
++ COL_BROWN=
++ COL_LIGHT_GRAY=
++ COL_DARK_GRAY=
++ TICK='[✓]'
++ CROSS='[✗]'
++ INFO='[i]'
++ QST='[?]'
++ DONE=' done!'
++ OVER='\r'
+ source /etc/.pihole/advanced/Scripts/database_migration/gravity-db.sh
++ readonly scriptPath=/etc/.pihole/advanced/Scripts/database_migration/gravity
++ scriptPath=/etc/.pihole/advanced/Scripts/database_migration/gravity
+ basename=pihole
+ PIHOLE_COMMAND=/usr/local/bin/pihole
+ piholeDir=/etc/pihole
+ whitelistFile=/etc/pihole/whitelist.txt
+ blacklistFile=/etc/pihole/blacklist.txt
+ regexFile=/etc/pihole/regex.list
+ adListFile=/etc/pihole/adlists.list
+ localList=/etc/pihole/local.list
+ VPNList=/etc/openvpn/ipp.txt
+ piholeGitDir=/etc/.pihole
+ gravityDBfile_default=/etc/pihole/gravity.db
+ GRAVITYDB=/etc/pihole/gravity.db
+ gravityDBschema=/etc/.pihole/advanced/Templates/gravity.db.sql
+ gravityDBcopy=/etc/.pihole/advanced/Templates/gravity_copy.sql
+ domainsExtension=domains
+ curl_connect_timeout=10
+ setupVars=/etc/pihole/setupVars.conf
+ [[ -f /etc/pihole/setupVars.conf ]]
+ source /etc/pihole/setupVars.conf
++ WEBPASSWORD=42eb7a0bcc263ff4903c0d2608fbaf11499e8c095feb9373d45b1d37b58ad424
++ DNSMASQ_LISTENING=all
++ DNSSEC=false
++ PIHOLE_INTERFACE=eth0
++ PIHOLE_DNS_1=185.228.168.10
++ PIHOLE_DNS_2=185.228.169.11
++ QUERY_LOGGING=false
++ INSTALL_WEB_SERVER=true
++ INSTALL_WEB_INTERFACE=true
++ LIGHTTPD_ENABLED=true
++ BLOCKING_ENABLED=true
++ CACHE_SIZE=10000
++ DNS_FQDN_REQUIRED=true
++ CONDITIONAL_FORWARDING_REVERSE=
++ DNS_BOGUS_PRIV=false
++ REV_SERVER=
++ REV_SERVER_DOMAIN=
++ REV_SERVER_TARGET=
++ REV_SERVER_CIDR=
++ TEMPERATUREUNIT=C
++ API_EXCLUDE_DOMAINS=
++ API_EXCLUDE_CLIENTS=
++ API_QUERY_LOG_SHOW=all
++ API_PRIVACY_MODE=false
+ : /tmp
+ '[' '!' -d /tmp ']'
+ '[' '!' -w /tmp ']'
+ pihole_FTL=/etc/pihole/pihole-FTL.conf
+ [[ -f /etc/pihole/pihole-FTL.conf ]]
+ source /etc/pihole/pihole-FTL.conf
++ BLOCKINGMODE=NULL
++ CNAME_DEEP_INSPECT=true
++ BLOCK_ESNI=true
++ EDNS0_ECS=true
++ RATE_LIMIT=1000/60
++ LOCAL_IPV4=
++ LOCAL_IPV6=
++ BLOCK_IPV4=
++ BLOCK_IPV6=
++ REPLY_WHEN_BUSY=DROP
++ MOZILLA_CANARY=true
++ BLOCK_TTL=2
++ BLOCK_ICLOUD_PR=true
++ MAXLOGAGE=24.0
++ PRIVACYLEVEL=0
++ IGNORE_LOCALHOST=no
++ AAAA_QUERY_ANALYSIS=
++ yes
```

Why is `yes` on its own line towards the end of `pihole-FTL.conf`?
Oh, there was a spurious space between the equals sign and the `yes` value on the
previous line. That explains that.

+ https://github.com/pi-hole/pi-hole/issues/4422#issuecomment-989035923
+ [How do I write standard error to a file while using "tee" with a pipe?](https://stackoverflow.com/a/692407)



## Some sanity checks

```
# pihole status
  [✓] FTL is listening on port 53
     [✓] UDP (IPv4)
     [✓] TCP (IPv4)
     [✓] UDP (IPv6)
     [✓] TCP (IPv6)

  [✓] Pi-hole blocking is enabled
```

```
# pihole version
  Pi-hole version is v5.17.3 (Latest: v5.17.3)
  web version is v5.21 (Latest: v5.21)
  FTL version is v5.25.1 (Latest: v5.25.1)
```

Just running `pihole -r`, which produces normal output until a certain point where
it goes haywire and prints tens of thousands of lines containing only the letter `y`
in a matter of seconds (no wonder the disk is filled).

```
root@fmj-dns:~
# pihole -r | tee pihole.log
```

The `y`'s start immediately after restarting the `pihole-FTL` service.

The systemd log (some metadata fields on each line have been stripped for legibility):
```
# journalctl -u pihole-FTL.service
fmj-dns systemd[1]: Starting Pi-hole FTL...
fmj-dns systemd[1]: Started Pi-hole FTL.
fmj-dns pihole-FTL[27249]: [00:18:05.358] Using log file /var/log/pihole/FTL.log
fmj-dns pihole-FTL[27249]: [00:18:05.358] ########## FTL started on fmj-dns! ##########
fmj-dns pihole-FTL[27249]: [00:18:05.358] FTL branch: master
fmj-dns pihole-FTL[27249]: [00:18:05.358] FTL version: v5.25.1
fmj-dns pihole-FTL[27249]: [00:18:05.358] FTL commit: 1c2257be
fmj-dns pihole-FTL[27249]: [00:18:05.358] FTL date: 2024-02-20 20:02:36 +0100
fmj-dns pihole-FTL[27249]: [00:18:05.358] FTL user: pihole
fmj-dns pihole-FTL[27249]: [00:18:05.359] Compiled for armv7hf (compiled on CI) using arm-linux-gnueabihf-gcc (Debian 8.3.0-2) 8.3.0
fmj-dns pihole-FTL[27249]: [00:18:05.359] Starting config file parsing (/etc/pihole/pihole-FTL.conf)
fmj-dns pihole-FTL[27249]: [00:18:05.359]    SOCKET_LISTENING: only local
fmj-dns pihole-FTL[27249]: [00:18:05.359]    AAAA_QUERY_ANALYSIS: Show AAAA queries
fmj-dns pihole-FTL[27249]: [00:18:05.359]    MAXDBDAYS: max age for stored queries is 60 days
fmj-dns pihole-FTL[27249]: [00:18:05.359]    RESOLVE_IPV6: Resolve IPv6 addresses
fmj-dns pihole-FTL[27249]: [00:18:05.359]    RESOLVE_IPV4: Resolve IPv4 addresses
fmj-dns pihole-FTL[27249]: [00:18:05.359]    DBINTERVAL: saving to DB file every minute
fmj-dns pihole-FTL[27249]: [00:18:05.360]    DBFILE: Using /etc/pihole/pihole-FTL.db
fmj-dns pihole-FTL[27249]: [00:18:05.360]    MAXLOGAGE: Importing up to 24.0 hours of log data
fmj-dns pihole-FTL[27249]: [00:18:05.360]    PRIVACYLEVEL: Set to 0
fmj-dns pihole-FTL[27249]: [00:18:05.360]    IGNORE_LOCALHOST: Show queries from localhost
fmj-dns pihole-FTL[27249]: [00:18:05.360]    BLOCKINGMODE: Null IPs for blocked domains
fmj-dns pihole-FTL[27249]: [00:18:05.360]    ANALYZE_ONLY_A_AND_AAAA: Disabled. Analyzing all queries
fmj-dns pihole-FTL[27249]: [00:18:05.360]    DBIMPORT: Importing history from database
fmj-dns pihole-FTL[27249]: [00:18:05.360]    PIDFILE: Using /run/pihole-FTL.pid
fmj-dns pihole-FTL[27249]: [00:18:05.361]    SOCKETFILE: Using /run/pihole/FTL.sock
fmj-dns pihole-FTL[27249]: [00:18:05.361]    SETUPVARSFILE: Using /etc/pihole/setupVars.conf
fmj-dns pihole-FTL[27249]: [00:18:05.361]    MACVENDORDB: Using /etc/pihole/macvendor.db
fmj-dns pihole-FTL[27249]: [00:18:05.361]    GRAVITYDB: Using /etc/pihole/gravity.db
fmj-dns pihole-FTL[27249]: [00:18:05.361]    PARSE_ARP_CACHE: Active
fmj-dns pihole-FTL[27249]: [00:18:05.361]    CNAME_DEEP_INSPECT: Active
fmj-dns pihole-FTL[27249]: [00:18:05.361]    DELAY_STARTUP: No delay requested.
fmj-dns pihole-FTL[27249]: [00:18:05.361]    BLOCK_ESNI: Enabled, blocking _esni.{blocked domain}
fmj-dns pihole-FTL[27249]: [00:18:05.361]    NICE: Set process niceness to -10 (default)
fmj-dns pihole-FTL[27249]: [00:18:05.361]    MAXNETAGE: Removing IP addresses and host names from network table after 60 days
fmj-dns pihole-FTL[27249]: [00:18:05.362]    NAMES_FROM_NETDB: Enabled, trying to get names from network database
fmj-dns pihole-FTL[27249]: [00:18:05.362]    EDNS0_ECS: Overwrite client from ECS information
fmj-dns pihole-FTL[27249]: [00:18:05.362]    REFRESH_HOSTNAMES: Periodically refreshing IPv4 names
fmj-dns pihole-FTL[27249]: [00:18:05.362]    RATE_LIMIT: Rate-limiting client making more than 1000 queries in 60 seconds
fmj-dns pihole-FTL[27249]: [00:18:05.362]    LOCAL_IPV4: Automatic interface-dependent detection of address
fmj-dns pihole-FTL[27249]: [00:18:05.362]    LOCAL_IPV6: Automatic interface-dependent detection of address
fmj-dns pihole-FTL[27249]: [00:18:05.362]    BLOCK_IPV4: Automatic interface-dependent detection of address
fmj-dns pihole-FTL[27249]: [00:18:05.362]    BLOCK_IPV6: Automatic interface-dependent detection of address
fmj-dns pihole-FTL[27249]: [00:18:05.362]    SHOW_DNSSEC: Enabled, showing automatically generated DNSSEC queries
fmj-dns pihole-FTL[27249]: [00:18:05.362]    MOZILLA_CANARY: Enabled
fmj-dns pihole-FTL[27249]: [00:18:05.362]    PIHOLE_PTR: internal PTR generation enabled (pi.hole)
fmj-dns pihole-FTL[27249]: [00:18:05.363]    ADDR2LINE: Enabled
fmj-dns pihole-FTL[27249]: [00:18:05.363]    REPLY_WHEN_BUSY: Drop queries when the database is busy
fmj-dns pihole-FTL[27249]: [00:18:05.363]    BLOCK_TTL: 2 seconds
fmj-dns pihole-FTL[27249]: [00:18:05.363]    BLOCK_ICLOUD_PR: Enabled
fmj-dns pihole-FTL[27249]: [00:18:05.363]    CHECK_LOAD: Enabled
fmj-dns pihole-FTL[27249]: [00:18:05.363]    CHECK_SHMEM: Warning if shared-memory usage exceeds 90%
fmj-dns pihole-FTL[27249]: [00:18:05.363]    CHECK_DISK: Warning if certain disk usage exceeds 90%
fmj-dns pihole-FTL[27249]: [00:18:05.364] F
```

Apart from that trailing `F` I see nothing unusual.



## Links and notes

+ https://docs.pi-hole.net/core/pihole-command
+ https://discourse.pi-hole.net/t/the-pihole-command-with-examples/738
+ https://stackoverflow.com/questions/418896/how-to-redirect-output-to-a-file-and-stdout
