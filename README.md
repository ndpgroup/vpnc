vpnc
======

#### A VPN client compatible with Cisco's EasyVPN equipment.

Supports IPSec (ESP) with Mode Configuration and Xauth.  Supports only
shared-secret IPSec authentication with Xauth, AES (256, 192, 128),
3DES, 1DES, MD5, SHA1, DH1/2/5 and IP tunneling.

It runs entirely in userspace. Only "Universal TUN/TAP device
driver support" is needed in kernel.

[Project home page](http://www.unix-ag.uni-kl.de/~massar/vpnc/)


Contents of this file
----------------------------------------------------------------------------


- General configuration of vpnc
- Using a modified script
- Additional steps to configure hybrid authentication
- Setting up vpnc on Vista 64bit
- Known problems


General configuration of vpnc
----------------------------------------------------------------------------


Required Libraries:

- [libgcrypt](https://www.gnu.org/software/libgcrypt/) (version `1.1.90` for `0.2-rm+zomb-pre7` or later)
- libopenssl (optional, to provide hybrid support)

It reads configuration data from the following places:

- From command-line options
- From config file(s) specified on the command line
- From `/etc/vpnc/default.conf`  only if no configfile was given on the command line
- From `/etc/vpnc.conf`          same as `default.conf`, ie: both are used, or none
- If a setting is not given in any of those places, it prompts the user.

The configuration information it currently needs is:

```shell
     Option Config file item
  --gateway IPSec gateway
       --id IPSec ID
(no option) IPSec secret
 --username Xauth username
(no option) Xauth password
```

A sample configuration file is:

```
# This is a sample configuration file.
IPSec gateway 127.0.0.1
IPSec ID laughing-vpn
IPSec secret hahaha
Xauth username geoffk
```

Note that all strings start exactly one space after the keyword
string, and run to the end of the line.  This lets you put any kind of
weird character (except `CR`, `LF` and `NUL`) in your strings, but it
does mean you can't add comments after a string, or spaces before them.

It may be easier to use the `--print-config` option to generate the
config file, and then delete any lines (like a password) that you want
to be prompted for.

If you don't know the Group ID and Secret string, ask your
administrator. If (s)he declines and refers to the
configuration files provided for the vpnclient program, tell
him/her that the contents of that files is (though scrambled)
not really protected. If you have a working configuration file
(`.pcf` file) for the Cisco client then you can use the `pcf2vpnc`
utility instead, which will extract most/all of the required
information and convert it into a vpnc configuration file.


Using a modified script
----------------------------------------------------------------------------

Please note that vpnc itself does NOT setup routing. You need to do this
yourself, or use `--script "Script"` in the config file.
The default script is /etc/vpnc/vpnc-script which sets a default route
to the remote network, or if the Concentrator provided split-network
settings, these are used to setup routes.

This option is passed to system(), so you can use any shell-specials you
like. This script gets called three times:
- `$reason == pre-init`: this is before vpnc opens the tun device
   so you can do what is necessary to ensure that it is available.
   Note that none of the variables mentioned below is available
- `$reason == connect`: this is what used to be "Config Script".
   The connection is established, but vpnc will not begin forwarding
   packets until the script finishes.
- `$reason == disconnect`: This is called just after vpnc received a signal.
   Note that vpnc will not forward packets anymore while the script is
   running or thereafter.

Information is passed from vpnc via environment variables:

```
#* reason                       -- why this script was called, one of: pre-init connect disconnect
#* VPNGATEWAY                   -- vpn gateway address (always present)
#* TUNDEV                       -- tunnel device (always present)
#* INTERNAL_IP4_ADDRESS         -- address (always present)
#* INTERNAL_IP4_NETMASK         -- netmask (often unset)
#* INTERNAL_IP4_DNS             -- list of dns servers
#* INTERNAL_IP4_NBNS            -- list of wins servers
#* CISCO_DEF_DOMAIN             -- default domain name
#* CISCO_BANNER                 -- banner from server
#* CISCO_SPLIT_INC              -- number of networks in split-network-list
#* CISCO_SPLIT_INC_%d_ADDR      -- network address
#* CISCO_SPLIT_INC_%d_MASK      -- subnet mask (for example: 255.255.255.0)
#* CISCO_SPLIT_INC_%d_MASKLEN   -- subnet masklen (for example: 24)
#* CISCO_SPLIT_INC_%d_PROTOCOL  -- protocol (often just 0)
#* CISCO_SPLIT_INC_%d_SPORT     -- source port (often just 0)
#* CISCO_SPLIT_INC_%d_DPORT     -- destination port (often just 0)
```

Currently vpnc-script is not directly configurable from configfiles.
However, a workaround is to use a "wrapper-script" like this, to
disable `/etc/resolv.conf` rewriting and setup a custom split-routing:

```
#!/bin/sh

# this effectively disables changes to /etc/resolv.conf
INTERNAL_IP4_DNS=

# This sets up split networking regardless
# of the concentrators specifications.
# You can add as many routes as you want,
# but you must set the counter $CISCO_SPLIT_INC
# accordingly
CISCO_SPLIT_INC=1
CISCO_SPLIT_INC_0_ADDR=131.246.89.7
CISCO_SPLIT_INC_0_MASK=255.255.255.255
CISCO_SPLIT_INC_0_MASKLEN=32
CISCO_SPLIT_INC_0_PROTOCOL=0
CISCO_SPLIT_INC_0_SPORT=0
CISCO_SPLIT_INC_0_DPORT=0

. /etc/vpnc/vpnc-script
```

Store this example script, for example in /etc/vpnc/custom-script,
do a `chmod +x /etc/vpnc/custom-script` and add
`Script /etc/vpnc/custom-script` to your configuration.


Additional steps to configure hybrid authentication
----------------------------------------------------------------------------

To use the hybrid extension add
	`Use Hybrid Auth`
to your .conf file or add
	`--hybrid`
when starting vpnc.

The trusted root certificate may be passed by adding
	`CA-File <root_certificate.pem>`
to your .conf file or adding
	`--ca-file <root_certificate.pem>`
when starting vpnc.

The trusted root certificate may be contained in a directory by adding
	`CA-Dir <trusted_certificate_directory>`
to your .conf file or adding
	`--ca-dir <trusted_certificate_directory>`
when starting vpnc.
The default is
	`/etc/ssl`

As the trusted certificate is referenced by the hash of the subject name,
the directory has to contain the certificate named like this hash_value.
A link can also be used like in `/etc/ssl/certs/`.
The hash value can be calculated by e.g:

```bash
openssl x509 -in <ca_certfile.pem> -noout -hash
```


Setting up vpnc on Vista 64bit
----------------------------------------------------------------------------

1. Install cygwin onto vista.  [Details here](http://www.cygwin.com/)
2. Make sure you install the development options for cygwin to give you
   access to make and gcc etc
3. Make sure you install libgcrypt for cygwin as it is needed in the make
4. Modify the bash.exe to run as administrator or you will have
   privilege issues later, this is done on the properties tab of the
   executable in `c:/cygwin/bin`
4. [Download the latest vpnc tarball](http://www.unix-ag.uni-kl.de/~massar/vpnc/)
5. Unzip and explode the tarball
6. modify `tap-win32.h` to change `#define TAP_COMPONENT_ID "tap0801"` to
   `"tap0901"` (No sure if this is necessary but I did it and it is working
   for me)
7. make
8. You should have a shiny new `vpnc.exe`
9. [Download openvpn](http://openvpn.net/download.html).  I used
   `openvpn-2.1_rc4-install.exe` as all other version I tried had errors
   during install
10. Run the exe but only install the *TAP-Win32 Adapter V9*
11. Go to control Panel | Network Connections and rename the *TAP* device
    to my-tap
12. create a `/etc/vpnc/default.conf` file something like this:

```
IPSec gateway YOURGATEWAY
IPSec ID YOURID
IPSec obfuscated secret YOURREALYLONGHEXVALUE (you can use your clear
text password here if you remove obfuscated)
Xauth username YOURUSERNAME
Xauth password YOURPASSWORD
Interface name my-tap
Interface mode tap
Local Port 0
```

See the general config section above and the manpage for details.


Known problems
----------------------------------------------------------------------------

**Problem:**
In some environments it may happen that stuff works for a while and then
stops working.

**Reason:**
The dhcp leases are very short intervals and on each renew the dhcp
client overwrites things like `/etc/resolv.conf` and maybe the default route.

***Solution:***
Fix your dhcpclient. On Debian that problem can be fixed by installing
and using resolvconf to modify that file instead of modifying it directly.