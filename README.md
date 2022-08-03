# ansible-role-bind

This Ansible role will deploy a Docker container running the ISC Bind DNS
service and configure it to your liking.

> The container used is [`jonasal/bind`][9] from [here][10].



# Installation

This repository does not have any dependencies, so just move into your `roles/`
folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-bind_dns.git bind
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: dns_servers
  name: Install Bind DNS server
  roles:
    - bind
```



# Usage

Bind has a ton of configuration possibilities so it is absolutely impossible
for me to create a role that covers every conceivable usecase, but I have tried
to make the most common ones easy to achieve and some more advanced stuff
possible by allowing freestyle text in the variables available.

I like [this][1] for reading about what options are available, but even that
does not cover everything.


## Example Configuration

To add some kind of structure to the example configuration I will split it into
four different categories.

### Container Options
The settings grouped in the Container options section have more of an impact on
the host and the actual Docker container than the Bind program. One setting
that probably should be changed is the base directory in which the container
will find the configuration files and write its logs:

```yaml
bind_container_base_dir: "/srv/bind"
```

Then there is the following setting:

```yaml
bind_use_host_network: false
```

which, when set to `true`, make the container use the `host` network (and thus
ignore anything listed in `bind_container_networks` and `bind_container_ports`).
This will most likely be necessary in case you want to enable IPv6 support,
but please read [this section][8] before you do.

> Don't forget to update `bind_container_command` and `bind_listen_on_v6` if you
> do enable IPv6.

### Bind Logging
Logging is important, and Bind is not very good at it. By default the container
is configured so that Bind tries to load its logging configuration from a file,
but if anything fails during startup it will not output any information. The
only way around this is then to force all log output to `stderr` (i.e. the
console) by setting this.

```yaml
bind_log_force_stderr: true
```

When you know everything works you may turn this off again and load the proper
configuration from the
[`named.cong.logging`](./templates/bind/named.conf.logging.j2) file. [This][2]
resource is pretty good at explaining how the logging works, but important to
know here is that I have defined a default "console" channel which always try
to output to stderr. The severity of that one is controlled with

```yaml
bind_log_channel_console_severity: "info"
```

and the obligatory "default" category will have this "console" channel included
unless you manually change it to something else.

```yaml
bind_log_category_default: [ "console" ]
```

Beyond that you may define your own channels and categories like this

```yaml
bind_log_channels:
  "custom_channel_1":
    severity: "warning"
  "custom_channel_2":
    severity: "debug 1"
    print_time: no
    print_severity: no
    print_category: no

bind_log_categories:
  "client":
    - "custom_channel_1"
  "database":
    - "custom_channel_2"
    - "console"
```

### Bind Options
Looking in the [defaults](./defaults/main.yml) file, and then in the
[named.conf.options](./templates/bind/named.conf.options.j2) file, it should
be quite easy to see what is going on here.

Unless you try to do something very advanced you can actually get away with
the `options` block being pretty sparse. But, as was mentioned at the beginning,
there exists [a ton of options][3] here and there is no way to prepare for all
of them. As a workaround for this it is possible to add some "raw" text into
the options file like this.

```yaml
bind_options_raw: |
  custom_option_1 yes;
  custom_option_2 {
    "list_item";
  };
```

The `|` character here allows you to write a multiline text (with empty lines!),
so you just need to make sure that noting is indented less than the first line.


### Bind Local
It is in this file where most of the action will take place. Here you define
the controls and the zones you intend to serve. To begin with we take a look
at the [controls][4] setting and how you configure it.

#### bind_controls
```yaml
bind_controls:
  "name_of_key_1":
    inet: '*'
    port: '953'
    allow: [ 'localhost' ]
    secret: "abcdef"
  "name_of_key_2":
    secret: "ghijkl"
```

The name needs to be unique, which is why it is used as the key in the map
above. The only thing that is required is the `secrets` value, and that may
be created by following [these steps][5].

#### `bind_include_default_zones` and `bind_include_rfc1918`
Of the following two settings you should set the `rfc1918` one to `false` if
you want your Bind instance to be able to [respond][6] with IP addresses that
are in [private ranges][7].

```yaml
bind_include_default_zones: true
bind_include_rfc1918: true
```

#### bind_zones
Then we come to the largest variable in this entire role, as it allows you to
configure all your zones from a single location. Any value that does not have
a comment after it will default to the value written here.

```yaml
bind_zones:
  "example.com":
    type: "primary"  # Required
    allow_transfer: [ "none" ]
    also_notify: []
    ttl: "1h"
    refresh: "3h"
    retry: "1h"
    expire: "1w"
    neg_cache: "1h"
    nameservers:  # Must be at least 1 long if "primary"
      - "ns0.example.com"
    rname: "admin.example.com"
    hosts:
      "192.168.0.4":
        hostname: "ns0"
        cnames: [ "main-computer", "main" ]
      "192.168.0.5":
        hostname: "www"
    custom_records:
      "MX":
        - name: ""
          data: "10 mail.example.com."
      "SRV":
        - name: "_http._tcp.example.com."
          data: "0    5      80   www.example.com."
  "upstream.com":
    type: "secondary"  # Required
    primaries:  # Must be present in case of "secondary"
      - "10.0.0.4"
    raw_options: |
      zero-no-soa-ttl yes;
      zone-statistics full;
```

So while I can try to explain all of these settings I think it is easier to just
add the files outputted here instead so you can see how each value will be
printed.

> Also, read the section about [`rname`][11] to understand its formatting.

##### named.conf.local
```
zone "example.com" {
    type primary;
    file "/var/cache/bind/example.com/soa.conf";

    allow-transfer {
        none;
    };
};

zone "upstream.com" {
    type secondary;
    file "/var/cache/bind/db.upstream.com.secondary";
    primaries {
        10.0.0.4;
    };

    zero-no-soa-ttl yes;
    zone-statistics full;
};
```

##### example.com zonefile
```
$TTL 1h
$ORIGIN example.com.
@                   IN    SOA     ns0.example.com. admin.example.com. (
                          1658264049    ; serial
                          3h            ; refresh
                          1h            ; retry
                          1w            ; expire
                          1h            ; negative caching TTL
                          )
@                   IN    NS      ns0.example.com.

ns0                        IN  A      192.168.0.4
main-computer              IN  CNAME  ns0
main                       IN  CNAME  ns0
www                        IN  A      192.168.0.5

1w                         IN  MX      10 mail.example.com
_http._tcp.example.com.    IN  SRV     0    5      80   www.example.com.
```

[1]: https://bind9.readthedocs.io/en/latest/reference.html
[2]: https://www.zytrax.com/books/dns/ch7/logging.html
[3]: https://bind9.readthedocs.io/en/latest/reference.html#options-block-grammar
[4]: https://www.zytrax.com/books/dns/ch7/controls.html
[5]: https://github.com/JonasAlfredsson/docker-bind#create-rndckey
[6]: https://serverfault.com/a/306109
[7]: https://en.wikipedia.org/wiki/Private_network
[8]: https://github.com/JonasAlfredsson/docker-bind#docker-network-mode
[9]: https://hub.docker.com/r/jonasal/bind/tags
[10]: https://github.com/JonasAlfredsson/docker-bind
[11]: https://en.wikipedia.org/wiki/SOA_record
