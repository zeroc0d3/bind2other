# bind2other

From the named.conf, make the configuration of NSD, Unbound, dnsdist with equivalent movement as much as possible

  - [Internet Week 2016 DNSOPS.JP BoF presentation materials](http://dnsops.jp/bof/20161201/bind2other.pdf)

## How to use

```
./bind2other.py named.conf
```

Since nsd.conf, unbound.conf, dnsdist.conf can be created in the same directory, start nsd, unbound, dnsdist using them.

```
sudo nsd -c nsd.conf
sudo unbound -c unbound.conf
sudo dnsdist -C dnsdist.conf -d

dig @ 127.0.0.1 example.com      # Local zone
dig @ 127.0.0.1 www.google.com   # other than the local zone
```

## Requirements
  - Python 2.6
  - dnsdist 1.0.0
  - NSD4
  - Unbound

## What kind of configuration can be done?
  - Default [named.conf](./example/named.conf)
  - Generated config file
    - [dnsdist.conf](./example/dnsdist.conf)
    - [nsd.conf](./example/nsd.conf)
    - [unbound.conf](./example/unbound.conf)

### Basic Operation
```
Client --- (receive query) -> [dnsdist]
                                |
                                + - (qname is the local zone) ---> [NSD]
                                |
                                + - (except for the local zone) ---> [Unbound]
```
dnsdist listens on 0.0.0.0: 53, [::]: 53 to receive queries from clients.
When receiving DNS queries from clients,

  - NSD (127.0.0.1:40000) if local zone (zone written in named.conf)
  - If it is not a local zone, Unbound (127.0.0.1:40001)

Do forward.

The NSD reads (or transfers zones) zones written in nsd.conf as a normal authority server and listens for queries at 127.0.0.1:40000.

Unbound operates as a normal DNS resolver and listens for queries at 127.0.0.1:40001.

### Access restriction

  - Source IP queries that do not match allow - query are always REFUSED
  - For queries to the local zone, forward to the NSD. However, if the query type is AXFR / IXFR, allow / deny according to allow-transfer (options, or each zone).
  - Only queries of source IP that are not addressed to the local zone and match allow - recursion are forwarded to Unbound.

## Function of corresponding named.conf

Only a few functions of BIND 9 are supported
```
acl mynetwork1 { 10.0.0.0/8; 192.168.0.0/16; };
acl mynetwork2 { 192.0.2.1; };
acl ournetwork { mynetwork1; mynetwork2; };   # ACL nesting is also OK.
```
#### Denial of negation in ACL can not be used
```
# acl evil_in_the_internet { 0.0.0.0/0; ! 1.1.1.1; }; # "!" is not allowed
```

### options
```
options {
  directory "/ etc";
  allow-query {any;};                         # The default is ANY (same as BIND 9)
  allow-recursion {ournetwork;};              # The default is none (BIND 9 is localhost; localnets)
  allow-transfer {mynetwork1; 1.1.1.1;};      # The default is none (BIND 9 is any)
};
```

### zone

Only master and slave zones. You can only specify allow-transfer for access restriction.
```
zone "example.com" {
  type master;
  file "example.com.zone";
  allow-transfer {none;};                     # option to override allow-transfer
};

zone "example2.com" {
  type slave;
  masters {10.0.0.1; 192.0.2.1;};
  allow-transfer {mynetwork 2; 127.0.0.1;};   # option to override allow-transfer
};
```

### view

Only correspond to match-clients.

  - Default [named.conf](./example/view/named.conf)
  - Generated config file
    - [dnsdist.conf](./example/view/dnsdist.conf)
    - nsd / unbound config for internal view
      - [nsd_internal.conf](./example/view/nsd_internal.conf)
      - [unbound_internal.conf](./example/view/unbound_internal.conf)
    - nsd / unbound config for external view
      - [nsd_external.conf](./example/view/nsd_external.conf)
      - [unbound_external.conf](./example/view/unbound_external.conf)
    - nsd / unbound config for default view
      - [nsd.conf](./example/view/nsd.conf)
      - [unbound.conf](./example/view/unbound.conf)

#### starting method
Invoke dnsdist in dnsdist.conf, nsd / unbound with config corresponding to each view

```
sudo nsd -c nsd.conf
sudo nsd -c nsd_external.conf
sudo nsd -c nsd_internal.conf

sudo unbound -c unbound.conf
sudo unbound -c unbound_external.conf
sudo unbound -c unbound_internal.conf

sudo dnsdist -C dnsdist.conf -d
```
