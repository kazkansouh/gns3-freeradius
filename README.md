# freeradius for GNS3

Builds upon the
[freeradius/freeradius-server](https://hub.docker.com/r/freeradius/freeradius-server)
Alpine images to improve compatibility with
[GNS3](https://github.com/GNS3/) by adding mounts. Also sets the
startup command to attach to `stdout` with debugging information out
of the box.

That is, when running image in GNS3 the following directories are
persistent:

* `/root`
* `/etc/raddb`

The startup command is `radiusd -f -X` (run as foreground process with
full debugging).

### Baseline

The image is based off `freeradius/freeradius-server:3.0.17-alpine`.

## Usage in GNS3 with Cisco appliances

1. Import the [appliance
   file](https://github.com/kazkansouh/gns3-freeradius/blob/master/freeradius.gns3a)
   into GNS3.
2. Configure the server. Once image is started in GNS3 use an
   auxiliary console (found by right-clicking and selecting aux
   console) to modify and save the configuration files in
   `/etc/raddb/`. The image will need to be stopped/started for the
   config files to be re-read and changes to take effect.

   For example, to add a client and user, issue the following two
   commands in the docker image: (a) enable router to communuicate
   with RADIUS server, (b) add user with plain text password:

   ```bash
   $ cat - >> /etc/raddb/clients.conf << EOF
   client R1 {
          ipv4addr = 192.168.0.1
          proto = udp
          secret = secretcisco
          nas_type = cisco
   }
   EOF
   ```
   ```bash
   $ cat - >> /etc/raddb/users << EOF
   admin Cleartext-Password := "cisco"
          Service-Type = NAS-Prompt-User,
          Cisco-AVPair = "shell:priv-lvl=15"
   EOF
   ```

   For more information about configuring `freeradius` for Cisco
   devices see
   [here](https://www.cisco.com/c/en/us/support/docs/security-vpn/remote-authentication-dial-user-service-radius/116291-configure-freeradius-00.html).

3. On the Cisco router, issue:
   ```
   R1(config)#aaa new-model
   R1(config)#radius server MyRADIUS
   R1(config-radius-server)#$address ipv4 192.168.0.2 auth-port 1812 acct-port 1813
   R1(config-radius-server)#key secretcisco
   ```

   This can be tested with:
   ```
   R1#test aaa group radius server name MyRADIUS admin cisco legacy
   ```

4. When using radius for authentication and authorisation of device
   users, (i.e. by setting up authentication and authorisation lists
   to use radius) by default the console does not apply the
   authorisation. That is, it will not apply the below attribute when
   logging in via the console, so the user will need to manually issue
   `enable`.

   ```
   Cisco-AVPair = "shell:priv-lvl=15"
   ```

   This functionality can be enabled with the global directive:

   ```
   aaa authorization console
   ```

## Usage with Cisco Switch and 802.1X

Configure the radius server as normal, steps 1 through 3 above. When
adding users to `/etc/raddb/users` the attributes specified are not
needed. That is, it is possible to add a user as:

```bash
$ echo user1 Cleartext-Password := \"cisco\" >> /etc/raddb/users
```

Configuration of the switch varies. With IOSvL2 (20170321:233949), the
following is a minimal configuration for setting up `g0/1` as an
authenticator:

```
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#dot1x system-auth-control
Switch(config)#aaa new-model
Switch(config)#radius server myserver
Switch(config-radius-server)#address ipv4 10.0.0.2 auth-port 1812
Switch(config-radius-server)#key secretcisco
Switch(config-radius-server)#exit
Switch(config)#aaa authentication dot1x default group radius
Switch(config)#int g0/1
Switch(config-if)#sw m a
Switch(config-if)#dot1x pae authenticator
Switch(config-if)#authentication port-control auto
```

The client connecting to `g0/1` needs to run `wpa_supplicant`. If
using `docker` in GNS3, select a docker image that has
`wpa_supplicant` installed
(e.g. [simple-client](https://cloud.docker.com/u/karimkanso/repository/docker/karimkanso/simple-client)). Then
update the configuration for the image in GNS3 to execute
`wpa_supplicant` when `eth0` comes up:

```
auto eth0
iface eth0 inet static
    address 192.168.0.2
    netmask 255.255.255.0
    gateway 192.168.0.1
    pre-up wpa_supplicant -Dwired -ieth0 -c/root/wpa.conf &
```

Here the file `/root/wpa.conf` contains the credentials for the
supplicant and should be present in the docker image. If using the
*simple-client* image, the `/root/` directory is persistent in GNS3,
so the file can be manually created as follows:

```bash
$ cat - > /root/wpa.conf << EOF
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0
eapol_version=2
ap_scan=0
network={
        key_mgmt=IEEE8021X
        eap=PEAP
        identity="user1"
        password="cisco"
        phase2="autheap=MSCHAPV2"
        eapol_flags=0
}
EOF
```

Some of these parameters can be tweaked, e.g. its possible to also use
`TTLS` instead of `PEAP`.

## Downloadable ACL (dACL)

It appears that running within GNS3 the Cisco IOSvL2 images do not
correctly support downloadable ACLs. However, its possible to
configure `freeradius` to serve the correct attributes for use against
a real switch.

1. When creating the users, add the desired attributes that define the
   ACL.

   1. Directly specify the ACL lines using `inacl#<n>`. E.g.

        ```
        user1 Cleartext-Password := "cisco"
            Cisco-AVPair += "ip:inacl#10=permit icmp any any",
            Cisco-AVPair += "ip:inacl#20=permit tcp host 10.1.0.130 host 10.1.0.132 eq 80",
            Cisco-AVPair += "ip:inacl#30=deny ip any any"
        ```
   2. Or, specify the name of an ACL defined on the switch using
      `Filter-Id`. E.g.

        ```
        user2 Cleartext-Password := "cisco"
            Filter-Id = "PingOK.in"

        ```

      Here, the `.in` applies the `PingOK` ACL on the ingress of the
      port. It could have been `.out` instead to apply on egress.

2. Due to an implementation detail of `freeradius`, its required to
   update the `post-auth` section of the site configuration to
   re-include the needed authorize section so the attributes are
   placed into the Accept message. See
   [here](http://lists.freeradius.org/pipermail/freeradius-devel/2013-July/008457.html)
   for more info. Thus, assuming a default setup, its needed to insert
   `files.authorize` into the `post-auth` section of the file
   `/etc/raddb/sites-enabled/default`. E.g.

    ```
    post-auth {
        files.authorize
        ...
    }
    ```

3. Configure the switch to accept network authorisation (in addition
   to 802.1X authentication). This is done by configuring `aaa
   authorization network default group radius` along with assigning a
   default ACL to the interfaces.
