# policyd-asn

A Postfix policy daemon that blocks email from specific Autonomous System Numbers (ASNs). The most common reason one might want to implement this is to combat spam.

## What It Does

This script integrates with Postfix as a policy service to reject incoming mail based on the sender's ASN. It looks up the client's IPv4 or IPv6 address in the MaxMind GeoLite2 ASN database and checks it against a user-created blocklist of ASNs.

## How It Works

1. Postfix sends client connection details to the policy daemon
2. The script extracts the client's IP address
3. Looks up the ASN using `mmdblookup`
4. Checks if the ASN exists in the blocklist via `grep`
5. Returns either `REJECT` (blocked) or `DUNNO` (allowed) to Postfix

Using a local database seemed preferable to making remote API calls, whois, dig, or other methods for every incoming email. It should be trival to swap out the MaxMind database with your preferred provider should you wish to do so. However, MaxMind is the most widely supported and usually already being used for other services (i.e. Nginx or Apache).

## Requirements

- Postfix mail server
- `mmdblookup` (from MaxMind GeoIP2 tools)
- GeoLite2 ASN database
- `awk`, `grep` (standard POSIX utilities)

## Installation

1. Install MaxMind GeoIP tools & databases (most distros have appropiate packages). Highly recommended to configure their updater service too.

2. Copy the `policyd-asn` script to your system, preferably located at `/usr/local/bin/` and making sure it is executable.

3. Create your ASN blocklist:
   `touch /etc/postfix/client_asn_access`
   
   Add one ASN per line (numbers only):
   ```
   # Comments are okay but make sure they are on their own line
   12345
   67890
   ```

5. Configure Postfix by editing `/etc/postfix/master.cf` and appending at the very bottom:
   ```
   # Reject bad ASNs
   policyd-asn  unix  -       n       n       -       0       spawn
       user=nobody argv=/usr/local/bin/policyd-asn
   ```

6. Add the policy service to your `/etc/postfix/main.cf`:
   ```
   smtpd_client_restrictions = permit_mynetworks,
       reject_unknown_client_hostname,
       **check_policy_service unix:private/policyd-asn**
   ```

7. Reload Postfix

## Configuration

Edit the script to customize these paths if needed:

- `DB_ASN` - Path to GeoLite2 ASN database (default: `/var/lib/GeoIP/GeoLite2-ASN.mmdb`)
- `ASN_BLOCKLIST` - Path to ASN blocklist file (default: `/etc/postfix/client_asn_access`)

## Finding ASNs to Block

You can look up ASNs for spam sources using:

```
mmdblookup --file /var/lib/GeoIP/GeoLite2-ASN.mmdb --ip <IP_ADDRESS> autonomous_system_number
```

Or use online tools like:
- https://www.radb.net/query
- https://www.peeringdb.com/
- https://bgp.he.net/

## Troubleshooting

It's just a bash script, add some debug logging like this after the input reading loop:

`echo "DEBUG: client_address=$client_address CLIENT_ASN=$CLIENT_ASN" >> /tmp/policyd-debug.log`

Check `/var/log/mail.log` for rejection messages.

## License

MIT License - feel free to use and modify as needed.
