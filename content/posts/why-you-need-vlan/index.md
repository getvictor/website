+++
title = "What is a VLAN and why you need it in your home network"
description = "VLANs improve network security, but require configuration on your router, switch, and access points"
authors = ["Victor Lyuboslavsky"]
image = "vlan-house.png"
date = 2024-11-27
categories = ["DevOps & Infrastructure"]
tags = ["Networking", "Cyber Security"]
draft = false
+++

## What is a VLAN?

VLAN (Virtual Local Area Network) technology allows you to create multiple isolated networks on a single physical
network. For example, a single ethernet wire or a WLAN (wireless LAN) can support multiple VLANs. VLANs improve network
security, performance, and scalability.

## How does VLAN improve network security?

VLAN improves network security by isolating devices into separate networks. This isolation prevents devices in one VLAN
from communicating with devices in another VLAN. For example, you can create a separate VLAN for your IoT (Internet of
Things) devices, such as smart light bulbs and thermostats, to prevent them from accessing your primary network. You can
also create a separate VLAN for guest devices to prevent them from accessing your main network and other VLANs.

{{< figure src="VLAN-basic.png" alt="Router with three VLANS -- office, IoT, and guest.">}}

For example, if a hacker gains access to a device in the IoT VLAN, they won't be able to access devices in the office
VLAN or the guest VLAN. This isolation limits the damage that a hacker can do to your network.

## How does VLAN work?

VLAN works by adding a VLAN tag to each network packet. The VLAN tag contains the VLAN ID, which identifies the VLAN to
which the packet belongs. Network switches use the VLAN tag to forward packets only to devices in the same VLAN. Routers
can route packets between different VLANs based on their VLAN tags.

## How to set up VLANs in your home network

Unfortunately, setting up a VLAN in your home network is not as simple as flipping a switch. Multiple parts of your home
network need to be configured, and some older or cheaper hardware, such as no-configuration network switches, may not
support VLANs.

{{< figure src="VLAN-full.png" alt="Detailed picture of router with three VLANS -- office, IoT, and guest. The picture includes firewall and switches.">}}

### Selecting VLAN tags and IP ranges

Before configuring VLANs, decide how many VLAN tags you need and what each tag will represent. Also, determine what IP
ranges will map to each VLAN. Some people map the VLAN ID to the third octet of the IP address. For example, VLAN 333
may use the IP range 10.0.333.0/24.

On the other hand, there is a security argument for using random VLAN IDs. If a hacker gets access to your network, they
won't know what each VLAN ID represents and may even have difficulty figuring out which VLAN IDs are active. This
security approach is often referred to as security through obscurity.

Some common VLANs are:

- Office
- IoT
- Guest
- Media

### Router interfaces

The network router interface is the first place to configure VLANs. A router interface is a physical or virtual router
port connecting to a network. So, you must configure the router interface to support your VLAN-selected tags.

### DHCP

Dynamic Host Configuration Protocol (DHCP) is a network protocol that automatically assigns IP addresses to devices on a
network.

You need to configure a DHCP server to assign IP addresses to devices in each VLAN. You can use the same DHCP server for
all VLANs but must configure it to assign IP addresses from different ranges for each VLAN.

### Firewall

You need to configure firewall rules for each VLAN to control what traffic is allowed in and out of the VLAN. For
example, you may not allow devices on the Guest VLAN to access devices on the other VLANs.

Below is an example of our firewall rules for the GUEST VLAN.

{{< figure src="vlan-firewall.png" alt="pfsense firewall rules for GUEST VLAN, which only allows access to GUEST VLAN.">}}

The rules allow access to our local DNS server to block inappropriate content. The rules block all private networking
IPs (as defined by RFC 1918) except the VLAN's subnet.

- 10.0.0.0/8 (10.0.0.0 - 10.255.255.255)
- 172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
- 192.168.0.0/16 (192.168.0.0 - 192.168.255.255)

### Network switch

You need to configure the VLAN tags for each port on the network switch. For example, you may configure port 1 to be
part of the Office VLAN and port 2 to be part of the IoT VLAN. A single port can be part of multiple VLANs, often
required for a wireless access point.

Below is an example of the network switch configuration for our GUEST VLAN.

{{< figure src="vlan-network-switch.png" alt="pfsense firewall rules for GUEST VLAN, which only allows access to GUEST VLAN.">}}

The tagged ports include three wireless access points and the router port.

### Wireless Access Point (WAP)

You need to configure the SSIDs (Service Set Identifiers) for each VLAN on your wireless access point. In our case, we
created a WLAN for each VLAN—Office, IoT, and Guest. Each WLAN is associated with a VLAN tag, which we set in the
advanced options of the WLAN configuration, as shown below.

{{< figure src="vlan-wlan-config.png" alt="WLAN priority tab that specifies the VLAN tag of 4.">}}

## Guest network

Before adding a VLAN for our guest network, we used the "Guest Mode" feature on our WAP (Wireless Access Point). This
feature was secure because it isolated guest devices from our primary network. However, the user experience for our
guests was terrible.

The guest network directed users to a [captive portal](https://en.wikipedia.org/wiki/Captive_portal) before granting
them Internet access. Some child guests could not access the captive portal due to parental device restrictions. Guests'
devices also had trouble reconnecting to the guest network on a subsequent visit -- they were not automatically
reconnected.

Switching to a VLAN-based guest network significantly improved the user experience.

## How to specify VLAN on a wired connection

A single wired ethernet connection may be part of multiple VLANs. You can connect your computer to different VLANs for
testing or security reasons.

On a wired connection, you can specify the VLAN ID in your device's network settings. For example, you can add a virtual
interface with a specific VLAN tag on macOS.

{{< figure src="vlan-macos-config.png" alt="macOS network settings with VLAN ID specified.">}}

## Debugging notes

While setting up our VLANs, we encountered the issue of our computer not getting an IP address from the DHCP. After
reviewing the settings on our router and switch, we found that our settings did not save for some reason. Make sure to
reload your settings after making changes to ensure they stick.

## Further reading

- In the past, we discussed [how to set up a virtual router](../setting-up-a-virtual-router/).
- We also covered [how to create an IPv6-only linux server](../create-ipv6-only-linux-server/).

- **[Securing Private Keys with TPM 2.0: A Developer’s Guide](../how-to-use-tpm/)** _A hands-on walkthrough of using TPM
  2.0 for hardware-backed key protection, featuring code examples and practical usage patterns._

## Watch us discuss what is a VLAN and why you need it in your home network

{{< youtube R8vq50uRxik >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
