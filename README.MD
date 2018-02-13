# configuration_nftables-firewall.nft

A reasonably comprehensive firewall configuration for nftables..

## Table of Contents

* [Installation](#installation)
* [Requirements](#requirements)
* [Usage](#usage)
* [How It Works](#how-it-works)
* [FAQ](#FAQ)

Additionally, you can read some other notes on key points here;
* [Considerations](#considerations)
* [Named Interface Performance](#named-interfaces)

## Installation

Important:
Please consider reading this before moving forward: [Considerations](#considerations)

To install, you can clone the repository into your `/etc/nftables.d/` folder, then include it in your parent
`/etc/nftables.conf` file.


    $ git clone git@github.com:dmblack/configuration_nftables-firewall.nft.git /etc/nftables.d/


Please then consider the below [requirements](#requirements), and [usage](#usage).

## Requirements

Optional: [See These Notes on Named Interfaces](#named-interfaces)

The below defines are expected, visible at a parent level to the script (Basically; define them somewhere, but
avoid using this script's defines file). For more information; see [Usage](#usage).

This configuration does expect visibility of the following defines;

```bash
# Network Interfaces
#   Loopback
define trusted_interfaces = lo
#   Ethernet, and Wireless
define untrusted_interfaces = { enp2s0f0, wlo1 }
#   Trusted named (not always available / enabled interfaces) - eg VPN
define trusted_interfaces_named = ppp0
#   Untrusted named (not always available / enabled interfaces) - eg Bluetooth
define untrusted_interfaces_named = bnep0
#   Global Interfaces (A collection of static interfaces)
define global_interfaces = { $trusted_interfaces , $untrusted_interfaces }
#   Global Interfaces (named - A collection of named interfaces)
define global_interfaces_named = { $trusted_interfaces_named , $untrusted_interfaces_named }
```

It's important to provide all of these defines, as they used to logically associate rules to interfaces. The only optional interfaces are 'named'

I like to overcomplicate my defines within defines -sigh-. Please consider my below example;

```bash
# Interfaces
# Untrusted
define nic_inet_lan = enp0s25
define nic_inet_wireless = wlo1
define nic_inet_bluetooth = bnep0
# The only interface I trust..
define nic_inet_vpn = ppp0

# Common
define nic_inet_lo = lo

# Networks
define trusted_interfaces = $nic_inet_lo
define untrusted_interfaces = { $nic_inet_lan , $nic_inet_wireless }
define global_interfaces = { $trusted_interfaces , $untrusted_interfaces }
define trusted_interfaces_named = $nic_inet_vpn
define untrusted_interfaces_named = $nic_inet_bluetooth
define global_interfaces_named = { $trusted_interfaces_named , $untrusted_interfaces_named }
```


## Usage

To clarify a little of the Requriements, my usual topology of these configurations includes a defines thats 
exposed at a higher level of the rule exection. This roughly looks like this;

```
# Is basically the entry point of the scripts
#   this file should include the /etc/nftables.d/defines.nft file, and the
/etc/nftables.d/configurations-enabled.nft files.
`/etc/nftables.conf`
# Is where all child exposed defines are setup.
`/etc/nftables.d/defines.nft`
# Is where all my child configurations are included.
#   this file should then include this projects entry point 
(/etc/nftables.d/configuration_nftables-firewall.nft/configuration_nftables-firewall.nft)
`/etc/nftables.d/configurations-enabled.nft`
```

This then allows this to be a little 'dynamic' in behavior.


## How It Works
The overall design / topology revolves around a 'global', 'trusted', and 'untrusted' ruleset. Interfaces are
listed as trusted, or untrusted. The same is configured for services. These are then associated using NFTables
VMAP's.

There's a global ruleset that allows management of interfaces and services at a global level (Such as allowing
http[s] on any interface.)

To combat the known issues of toggled interfaces and NFTables (such as ppp0 vpn), which is only available
whilst it's configured; trusted and untrusted also include a trusted\_named and untrusted\_named list.

An example of an untrusted service, ssp, looks like this;
(Obviously; the first definition is not necessary..)
```
define du_trusted_service_tcp_sspd = 1900
add element filter untrusted_services_tcp_in { $du_trusted_service_tcp_sspd . tcp : drop }
```

WIP: Migrate from 'drop' of outgoing connections to be a 'reject icmp admin-prohibited', however this appears
to be a restriction with vmaps?


## FAQ
(Mostly asked by myself, to myself)

* Why does this project exist?

I've had a few side projects that relate directly to NFTables. After migrating 
home firewall to the NFTables library (from IPTables), I've been working on a
modular like implementation of rules I can quickly include on all my
infrastructure.

Managing these with SCM was done because I'm silly!

Another project will be live soon, in which NFTables queues management of deny
rules to userspace, with an applet (... electron? wip?) that allows interactive
management of the initiator of connections (As, like usual; established / 
related are often allowed)

* Goals of this Project

Provide a 'modular' like experience in implementing rules using NFtables. I'm in
the process of reading a really good on scripting IPTables and NFTables, which
may have some changes to this.


## Considerations

Please consider; these rules are personally defined with best interests at heart. The repository is locked
down to myself only. That being said; I cannot be held personally accountable for any negative side-effects
of using these rules.

I have a long history in the IT industry. I endeavour to abide by best practice
, and other security policy where applicable.

My rule of heart, when it comes to anything like the above;
Least access is the right access!

If any of the above has concerned you, please also consider; some distributions release with default rulesets
that are completely open (allow incoming and outgoing entirely, on any interface, at any time)!

You can check this by initiating ` $ sudo nft list ruleset `

If you have no rules in each of your chains, you're pretty much entirely exposed anyway :P


## Named Interfaces

Yes. There's a performance loss on named interfaces and nftables. This is due to the way the system matches 
(string?) comparison for these names for EVERY rule that otherwise references them.

For this reason; I endeavour to keep named rules at the bottom of the 'chain', hoping for other matches to be
met before these are encountered.

Unfortunately; there is not any other way to implement these rules without other dependencies (batch, daemon)
like options. I do like the idea of having more of a 'script' like configuration moving forward, and this may
be incorporated - but to date; I have never found the performance loss with named interfaces even casual usage
identifiable.

(ToDo: Add reference for named interface performance concerns)
