# Structure #

A data packet on the wire is called a frame and consists of binary data. Data on Ethernet is transmitted most-significant octet first. Within each octet, however, the least-significant bit is transmitted first.[1](1.md)
The table below shows the complete Ethernet frame, as transmitted, for the payload size up to the MTU of 1500 octets.[1](note.md) Some implementations of Gigabit Ethernet (and higher speed ethernets) support larger frames, known as jumbo frames.

![https://armtutorial.googlecode.com/svn/trunk/image/ETHERNET_Frame-Structure.png](https://armtutorial.googlecode.com/svn/trunk/image/ETHERNET_Frame-Structure.png)

# Preamble and start frame delimiter #

A frame starts with a 7-octet preamble and 1-octet start frame delimiter (SFD).[3](note.md) Prior to Fast Ethernet, the on-the-wire bit pattern for this portion of the frame is 10101010 10101010 10101010 10101010 10101010 10101010 10101010 10101011.[3](3.md) Since octets are transmitted least-significant bit first the corresponding hexadecimal representation is 0x55 0x55 0x55 0x55 0x55 0x55 0x55 0xD5.
PHY transceiver chips used for Fast Ethernet feature a 4-bit (one nibble) Media Independent Interface. Therefore the preamble will consist of 14 instances of 0x5, and the start frame delimiter 0x5 0xD. Gigabit Ethernet transceiver chips use a Gigabit Media Independent Interface that works 8-bits at a time, and 10 Gbit/s (XGMII) PHY works with 32-bits at a time.

# Header #

The header features destination and source MAC addresses which have 6 octets each, the EtherType protocol identifier field and optional IEEE 802.1Q tag.

# 802.1Q tag #

The IEEE 802.1Q tag is an optional 4-octet field that indicates Virtual LAN (VLAN) membership and IEEE 802.1p priority.

# EtherType or length #

EtherType is a two-octet field in an Ethernet frame. It is used to indicate which protocol is encapsulated in the payload of an Ethernet Frame.

# Payload #

The minimum payload is 42 octets when 802.1Q tag is present and 46 octets when absent.[2](2.md)[4](note.md) and the maximum payload is 1500 octets. Non-standard jumbo frames allow for larger maximum payload size.

# Frame check sequence #

The frame check sequence is a 4-octet cyclic redundancy check which allows detection of corrupted data within the entire frame.

# Interframe gap #

Interframe gap is idle time between frames. After a frame has been sent, transmitters are required to transmit a minimum of 96 bits (12 octets) of idle line state before transmitting the next frame.


---

TCP-IP Stack for STM32

http://www.oryx-embedded.com/cyclone_tcp.html