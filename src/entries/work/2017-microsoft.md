---
title: Software Engineer
organization: Microsoft
organizationUrl: https://www.microsoft.com
location: Redmond, WA
start: 2017-10-30
---

At Microsoft, I work on the wireless networking and Miracast team. I have been
primarily focused on Miracast and the Wi-Fi Direct protocols in the Windows
operating system. My primary responsibilities include designing and implementing
new features, as well as ensuring that Windows maintains compatibility with the
broader Miracast ecosystem. I also help maintain Microsoft's extension to
Miracast, "Miracast over Infrastructure" (also called InfraCast).

My most recent project was to design and implement a backwards compatible method
of selecting which side of the Miracast connection should listen for the RTSP
connection. By default, the Miracast protocol specifies that the projecting
(source) device should listen for an incoming TCP connection on the Miracast
RTSP control port. This requires the projecting device to configure its firewall
to allow the inbound connection. That requirement can pose an issue when system
firewalls are configured to block all inbound connection requests. By allowing
the source device to request that the Miracast receiver should listen for the
RTSP TCP connection, the source device's firewall can continue blocking inbound
connection requests.

Previous projects include implementing an extension to the InfraCast protocol to
allow for a PIN to be required when setting up the connection. This feature
helps to ensure that the user is connecting to the intended receiver, such as in
an office building with multiple conference rooms, each with their own receiver.

I've also worked on some smaller projects for the recently announced Windows 10X
operating system. One was to ensure that the new operating system can be
certified by the Wi-Fi Alliance (WFA) for proper Wi-Fi operation by updating the
Microsoft toolkit used during the certification process. Another was to bring up
support for NFC hardware by bringing up the service required to interface with
the driver.
