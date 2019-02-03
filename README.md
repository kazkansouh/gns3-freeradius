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
   # cat - >> /etc/raddb/clients.conf << EOF
   client R1 {
          ipv4addr = 192.168.0.1
          proto = udp
          secret = secretcisco
          nas_type = cisco
   }
   EOF
   ```
   ```bash
   # cat - >> /etc/raddb/users << EOF
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
