# IoT Review
3. Be able to determine the type of message based on a hex dump of the data in a frame

- Suppose this is the hexdump of a message: b0 7f b9 05 b6 42 38 f9 d3 2c 81 80 08 06 00 01 08 00 06 04 00 02 38 f9 d3 2c 81 80 c0 a8 00 49 b0 7f b9 05 b6 42 c0 a8 00 01

          The best strategy is to keep the ethernet frame in mind when first reading the hexdump.
          - The first field in the ethernet frame is the dst mac addr: b0:7f:b9:05:b6:42 

          - The second field is the source mac addr: 38:f9:d3:2c:81:80

          - The third field is the protocol: 08 06 (arp)

          - Look up ARP protocol on wikipedia and begin to decipher hexdump accordingly

          - field 1: htype = 00 01

          - field 2: protocol type = ipv4 = 08 00

          - field 3: hw size = mac = 06
          
          - field 4: protocol size = ipv4_len = 04

          - field 5: opcode = reply = 00 02

          - field 6: sender mac: 38:f9:d3:2c:81:80

          - field 7: sender ip: c0 a8 00 49

          - field 8: target mac: b0:7f:b9:05:b6:42

          - field 9: target ip c0 a8 00 01

4. Know
- the operational states of DHCP

     - **INIT**: system boots and readies a DHCPDISCOVER routine. System moves into DISCOVER state.

     - **DISCOVER**: system initiates a timer to periodically send DHCPDISCOVER messages. System moves into selecting state.

     - **SELECTING**: system gathers offers from DHCP servers. System shoots one-shot timer which, upon expiration, reverts system back into DISCOVER state. Timer is stopped if selection criteria is met. System moves into REQUESTING state.

     - **REQUESTING**: system sends a DHCPREQUEST message to selected DHCP server requesting an IP address. System shoots a one-shot timer which, upon expiration, reverts system back into DISCOVER state. Timer is stopped if server ACKs system and system moves into TESTING_IP state. 

     - **TESTING_IP**: gratuitous (unwarranted) ARP is sent out to verify IP uniqueness. A 2 second oneshot timer is fired. If timer expires, move into BOUND state. If another system responds, send DHCPDECLINE message to server and revert to DHCPDISCOVER state.

     - **BOUND**: system is bound to an IP address for the alotted lease time given by the server. Oneshot timer routines are created at half of the lease time and 7/8ths of the lease time. If half of the lease time has passed, move to state RENEW. 

     - **RENEW**: system starts periodic timer that sends DHCPREQUEST messages to server that offered IP address. If server responds, revert to BOUND state. If 7/8ths of lease time has passed, move to REBIND state.

     - **REBIND**: system starts periodic timer that sends DHCPREQUEST message to any server to re-establish given IP address. If server responds, revert to BOUND state. Else, revert to DISCOVER state.

**NOTE**: all timer resets and restarts have been omitted from the operational states.

- the contents of the DHCP header
     - **OP**: 1 byte -> request == 0x01 ; reply == 0x02

     - **HTYPE**: hardware type -> 1 byte -> 10BASET == 0x01

     - **HLEN**: length HW addr -> 1 byte -> MAC_LEN == 0x06

     - **HOPS**: equivalent to ip->ttl -> 1 byte

     - **XID**: transaction id -> 4 bytes ; unique to each message

     - **SECS**: seconds elapsed since client began addr acquisition -> 2 bytes

     - **FLAGS**: broadcast == 0x8000 ; unicast == 0x0000 -> 2 bytes

     - **CIADDR**: client addr -> 4 bytes

     - **YIADDR**: your ip

     - **SIADDR**: server ip

     - **GIADDR**: gateway (router) ip

     - **CHADDR**: client HW addr with 192 bytes of padding -> 198 bytes

     - **MAGIC COOKIE**: 0x63825363

- the options fields:

     - **MESSAGE TYPE: 53**
          - Types of messages:

               - 1: DISCOVER: usual scheme is -> 53 01 01 255

               - 2: OFFER: scheme depends on server but usually returns offered IP along with lease time and identifying router

               - 3: REQUEST: usual scheme is -> 53 01 03 50 4 YIADDR 54 4 SIADDR 55 3 1 3 6 255

               - 4: DECLINE: scheme -> 53 01 04 255

               - 5: ACK: scheme depends on order of REQUEST message. Only sent by server

               - 6: NAK: scheme depends on order of request message. Only sent by server
               
               - 7: RELEASE: scheme -> 53 01 07 255

5. Be able to calculate the IP and UDP checksums of a DHCP message

- IP Checksum:
     - Suppose this is an ip hexdump: 4500 0034 0000 4000 4006 b8dc c0a8 0049 c0a8 004e


          Then, to calculate the IP checksum, do a 16-bit sum over each pair of octets and "roll over" any extraneous 1s. 

          0x4500 + 0x0034 = 0x4534

          0x4534 + 0x0000 = 0x4534

          0x4534 + 0x4000 = 0x8534

          0x8534 + 0x4006 = 0xc53a

          here, we **DON'T** sum the next pair of octets as it is the checksum. Skip to next pair

          0xc53a + 0xc0a8 = 0x185e2

          when a 16-bit overflow occurs, simply add the carry bit onto the LSb. So, 0x185e2 = 0x85e3

          0x85e3 + 0x0049 = 0x862c

          0x862c + 0xc0a8 = 0x146d4 -> 0x46d5

          0x46d5 + 0x004e = 0x4723

          Take the one's complement of 0x4723 and you have your checksum.

          ones_complement(0x4723) == 0xb8dc

          to verify, add 0x4723 and 0xb8dc and answer should be 0.

- TCP Checksum
     - Suppose this is the hexdump of a TCP message: e2f2 1f49 a321 3810 a11f 962c 8010 07fe (checksum field)c855 0000 0101 080a 0536 0054 001a 0a24 

     - Suppose this is the hexdump of the TCP's IP field: 4500 0034 0000 4000 4006 b8dc c0a8 0049 c0a8 004e


          Then, to compute the checksum, compute the checksum of a "pseudo-header" prior to adding all of these items.

          The pseudo-header consists of:

          the source ip address - c0a8 0049

          the destination ip address - c0a8 004e

          the transport protocol's data length - tcp calculated length

          the protocol field in the ip header and an 8-bit reserved field - 0600

          then, sum all the terms within the pseudo header.

          next, sum the result with all of the terms in the tcp message, take one's complement, and verify checksum.
