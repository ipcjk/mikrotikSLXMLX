# Mikrotik RouterOS vs Extreme SLX/MLX: BGP Attribute Flags Error

## Symptoms
When a BGP session is established between Mikrotik RouterOS > 7.x and Netiron and SLX products, unexpected termination or protocol messages occur.

## Cause

Mikrotik RouterOS > 7.x sends AS4_AGGREGATOR attributes flags (0xd0) with the extendend-length bit set
in the BGP UPDATE message, indicating and sending a 2-octet length field. 

This is not supported by Extreme products and leads to different behavior depending on the product.

Extreme MLX expects a 1-octet length field and terminates the BGP session with an "Attribute Flags Error" message.

Extreme SLX will accept a 2-octet length field and the flag but logs a warning message.

## RFC

RFC6793 says:

`The AS4_AGGREGATOR attribute in an UPDATE message SHALL be considered
malformed if the attribute length is not 8.
`

RFC4271 says:

`AGGREGATOR (Type Code 7)
AGGREGATOR is an optional transitive attribute of length 6.
The attribute contains the last AS number that formed the
aggregate route (encoded as 2 octets), followed by the IP
address of the BGP speaker that formed the aggregate route
(encoded as 4 octets).  This SHOULD be the same address as
the one used for the BGP Identifier of the speaker.
`

RFC4271 says over extended length:

`If the Extended Length bit of the Attribute Flags octet is set
to 0, the third octet of the Path Attribute contains the length
of the attribute data in octets.
         If the Extended Length bit of the Attribute Flags octet is set
         to 1, the third and fourth octets of the path attribute contain
         the length of the attribute data in octets.
`

RFC1771 (obsolete) says:
`The fourth high-order bit (bit 3) of the Attribute Flags octet
is the Extended Length bit.  It defines whether the Attribute
Length is one octet (if set to 0) or two octets (if set to 1).
Extended Length may be used only if the length of the attribute
value is greater than 255 octets.
` 

## Resolution
- Mikrotik RouterOS > 7.x shall not set the extended length bit for AGGREGATOR attribute flags
- Extreme products shall support the extended length bit for AGGREGATOR attribute flags

### Reproduce  with Mikrotik
- Configure 32-Bit ASN4 BGP session between Mikrotik RouterOS > 7.x and Extreme product 
### Reproduce  with FRR
- __Patch__ FRR forcing sending extended length bit for AGGREGATOR attribute flags
- Configure 32-Bit ASN4  BGP session between _patched_ FRR (see repository) and Extreme product

### MLX Behavior

#### Versions tested
    IronWare 6.2.0eT177

#### Symptoms
    BGP session flaps with "Attribute Flags Error" message.

#### show ip bgp neighbors last-packet-with-error
    Last error: received invalid AGGREGATOR attribute flag (0xd0)
    BGP4: 63 bytes hex dump of packet that contains error
    ffffffff ffffffff ffffffff ffffffff 003f0200
    00002440 01010050 02000602 01fa56ea 00400304
    0ae8b8fe 400600d0 070008fa 56ea0016 0ab8fe18
    0ae8bc49 100e6672 726f7574 696e676a 6f657267

#### Log
    Jul  1 14:46:21:N:BGP: Peer (VRF: default-vrf) XXXXXXXX DOWN (Attribute Flags Error)
    Jul  1 14:46:20:N:BGP: Peer (VRF: default-vrf) XXXXXXXX UP (ESTABLISHED)
    Jul  1 14:46:19:N:BGP: Peer (VRF: default-vrf) XXXXXXXX DOWN (Attribute Flags Error)
    Jul  1 14:46:18:N:BGP: Peer (VRF: default-vrf) XXXXXXXX UP (ESTABLISHED)
    Jul  1 14:46:18:N:BGP: Peer (VRF: default-vrf) XXXXXXXX DOWN (Attribute Flags Error)

### SLX Behavior

#### Versions tested
    SLX-OS 20.5.1

#### Symptoms
    BGP Session stays up 
    Routes are installed
    Log/Warning Entry is generated 

#### show ip bgp neighbors last-packet-with-error
    Last error: received invalid AGGREGATOR attribute flag (0xd0).
    BGP4: 0 bytes hex dump of packet that contains error

#### Log
    2023/07/01-13:56:34, [BGP-1005], 7486,, INFO, slx,  BGP: Neighbor XXXXXXXXon VRF default-vrf UP (ESTABLISHED).
    2023/07/01-13:56:35, [BGP-1002], 7487,, INFO, slx,  BGP received XXXXXXXX invalid AGGREGATOR attribute flag (0xd0).
