# WORKING WITH NETCONF (MANUALLY)

Network devices usually expose their management over CLI. 
**NETCONF protocol is an alternative to CLI and its main goal is to improve machine 
to machine communication** when managing the network. 
A typical use case would be an SDN controller automating network management. 
In order to achieve that, NETCONF :

* Defines a strict structure for configuration and operational data (usually with [YANG](https://tools.ietf.org/html/rfc6020.html) schemas)
* Adds transactional  processing of configuration changes for a single device but also for entire network
* Defines standardized set of (xml based) operations
* and more

**In this post we will take a look how we can quickly test the NETCONF
endpoint of a network device without any additional tools, just a Linux VM. 
We will connect to an XRv device over NETCONF and read its configuration.** 
This post does not present details of NETCONF itself, so make sure to get 
some basic info about NETCONF (nice overview: http://www.tail-f.com/what-is-netconf)

## Opening a netconf session

We have an XRv device by Cisco, version 6.1.2 with pre-configured NETCONF available. 
For details on how to enable NETCONF on XR, check [official documentation](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/data-models/guide/b-data-models-config-guide-asr9000/b-data-odels-config-guide-asr9000_chapter_01.html#id_20878).

Note on NETCONF transport: NETCONF relies on existing transport protocols such as SSH, plaintext TCP(telnet), BEEP etc. Since SSH is most commonly used, this guide will focus on NETCONF over SSH.
Typically, we would open CLI session with the device using following command:

```
ssh cisco@192.168.1.212
```

To access NETCONF, we just need to open SSH on port 830 (standard NETCONF server port) with subsystem **netconf**.

```
ssh cisco@192.168.1.212 -p 830 -s netconf
```
Just as with regular CLI, 

## Sending hello message

First order of business in a NETCONF session is to exchange HELLO messages.
Both the client side (user) and server side (device) are required to send HELLO message immediately after opening the session.
If everything went fine in the previous step, you should see XR's HELLO message:

```
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <capabilities>
  <capability>urn:ietf:params:netconf:base:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
  <capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
  <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
  <capability>http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-lib-cfg?module=Cisco-IOS-XR-aaa-lib-cfg&amp;revision=2015-11-09</capability>
  ...
  <capability>http://openconfig.net/yang/vlan?module=openconfig-vlan&amp;revision=2015-10-09&amp;deviation=cisco-xr-openconfig-vlan-deviations</capability>
 </capabilities>
 <session-id>3934303254</session-id>
</hello>
]]>]]>
```

We as a client need to send a simpler HELLO message to the server, so just copy and paste following xml into the window

```
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <capabilities>
        <capability>urn:ietf:params:netconf:base:1.0</capability>
        <capability>urn:ietf:params:netconf:base:1.1</capability>
    </capabilities>
</hello>
]]>]]>
```

**IMPORTANT: netconf:base:1.0 vs netconf:base:1.1 capability**

NETCONF can use 2 types of message separators.
The simple one identified by capability: utn:ietf:params:netconf:base:1.0 uses string of "]]>]]>" as message separator. Each message in this case needs to be followed by a ]]>]]> and a newline.

The second option when using  utn:ietf:params:netconf:base:1.1 capability,
is to frame the message using [chunk framing](https://tools.ietf.org/html/rfc6242#section-4.2) and we will use it with subsequent
messages.

Note on hello: No matter which framing mechanism is chosen, 
hello message needs to be ended with "]]>]]>" and a newline.
Note on XR: In this case we will be using netconf:base:1.1 and chunk framing
since XR only supports this type of chunking.

## Getting entire running configuration

At this point, the session is up and framing mechanism has been negotiated. Now we can send and receive RPCs defined by NETCONF protocol. A simple example would be getting the entire running configuration. In CLI we would perform:

```
RP/0/0/CPU0:PE2# show run
```

NETCONF version: using get-config RPC:

```
#170
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="101">
    <get-config>
        <source>
            <running/>
        </source>
    </get-config>
</rpc>
##
```

Which should result in an rpc-reply with similar content:

```
#495
<?xml version="1.0"?>
<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <aaa xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-aaa-locald-admin-cfg">
   <usernames>
    <username>
     <name>toor</name>
     <usergroup-under-usernames>
      <usergroup-under-username>
       <name>root-system</name>
      </usergroup-under-username>
     </usergroup-under-usernames>
     <secret>$1$FXDj$sTp2R26WyTkIagK8eA0HK/</secret>
    </username>
   </usernames>
  </aaa>

#371
  <crypto xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-crypto-sam-cfg">
   <ssh xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-crypto-ssh-cfg">
    <server>
     <v2></v2>
     <netconf-vrf-table>
      <vrf>
       <vrf-name>default</vrf-name>
       <enable></enable>
      </vrf>
     </netconf-vrf-table>
     <timeout>120</timeout>
    </server>
   </ssh>
  </crypto>

#1519
  <interface-configurations xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ifmgr-cfg">
   <interface-configuration>
    <active>act</active>
    <interface-name>MgmtEth0/0/CPU0/0</interface-name>
    <ipv4-network xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ipv4-io-cfg">
     <addresses>
      <primary>
       <address>192.168.1.212</address>
       <netmask>255.255.255.0</netmask>
      </primary>
     </addresses>
    </ipv4-network>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/0</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/1</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/2</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/3</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/4</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
   <interface-configuration>
    <active>act</active>
    <interface-name>GigabitEthernet0/0/0/5</interface-name>
    <shutdown></shutdown>
   </interface-configuration>
  </interface-configurations>

#220
  <ip-domain xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ip-domain-cfg">
   <vrfs>
    <vrf>
     <vrf-name>default</vrf-name>
     <name>demo.io</name>
     <lookup></lookup>
    </vrf>
   </vrfs>
  </ip-domain>

#355
  <ip xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ip-tcp-cfg">
   <cinetd>
    <services>
     <vrfs>
      <vrf>
       <vrf-name>default</vrf-name>
       <ipv4>
        <telnet>
         <tcp>
          <maximum-server>10</maximum-server>
         </tcp>
        </telnet>
       </ipv4>
      </vrf>
     </vrfs>
    </services>
   </cinetd>
  </ip>

#285
  <netconf-yang xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-man-netconf-cfg">
   <agent>
    <ssh>
     <enable></enable>
    </ssh>
   </agent>
  </netconf-yang>
  <host-names xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-shellutil-cfg">
   <host-name>PE2</host-name>
  </host-names>

#654
  <tty xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-tty-server-cfg">
   <tty-lines>
    <tty-line>
     <name>vty</name>
     <exec>
      <time-stamp>true</time-stamp>
      <timeout>
       <minutes>0</minutes>
       <seconds>0</seconds>
      </timeout>
     </exec>
    </tty-line>
    <tty-line>
     <name>console</name>
     <exec>
      <timeout>
       <minutes>0</minutes>
       <seconds>0</seconds>
      </timeout>
     </exec>
    </tty-line>
    <tty-line>
     <name>default</name>
     <exec>
      <timeout>
       <minutes>0</minutes>
       <seconds>0</seconds>
      </timeout>
     </exec>
    </tty-line>
   </tty-lines>
  </tty>

#232
  <vty xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-tty-vty-cfg">
   <vty-pools>
    <vty-pool>
     <pool-name>default</pool-name>
     <first-vty>0</first-vty>
     <last-vty>50</last-vty>
    </vty-pool>
   </vty-pools>
  </vty>

#3378
  <interfaces xmlns="http://openconfig.net/yang/interfaces">
   <interface>
    <name>MgmtEth0/0/CPU0/0</name>
    <config>
     <name>MgmtEth0/0/CPU0/0</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>true</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
    <subinterfaces>
     <subinterface>
      <index>0</index>
      <ipv4 xmlns="http://openconfig.net/yang/interfaces/ip">
       <address>
        <ip>192.168.1.212</ip>
        <config>
         <ip>192.168.1.212</ip>
         <prefix-length>24</prefix-length>
        </config>
       </address>
      </ipv4>
     </subinterface>
    </subinterfaces>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/0</name>
    <config>
     <name>GigabitEthernet0/0/0/0</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/1</name>
    <config>
     <name>GigabitEthernet0/0/0/1</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/2</name>
    <config>
     <name>GigabitEthernet0/0/0/2</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/3</name>
    <config>
     <name>GigabitEthernet0/0/0/3</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/4</name>
    <config>
     <name>GigabitEthernet0/0/0/4</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
   <interface>
    <name>GigabitEthernet0/0/0/5</name>
    <config>
     <name>GigabitEthernet0/0/0/5</name>
     <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:ethernetCsmacd</type>
     <enabled>false</enabled>
    </config>
    <ethernet xmlns="http://openconfig.net/yang/interfaces/ethernet">
     <config>
      <auto-negotiate>false</auto-negotiate>
     </config>
    </ethernet>
   </interface>
  </interfaces>
 </data>
</rpc-reply>

##

```

Note on XR response: Chunk framing mechanism allows a single NETCONF message to be split into multiple chunks, each chunk starting with a header of #371. The number specifies how many bytes follow. Each chunk needs to start with such header and at the end of entire message you should see section of "##" If we strip the headers and footers, we will end up with a simple XML message.

Note on get-config request: The request starts with header of #170, which means that the message contains 170 bytes of data. So when hand-crafting a NETCONF request with chunk framing, make sure to calculate number of characters and update the header. Text editors (such as Sublime text editor) should be able to show you number of characters in highlighted text. 


## Getting a subset of configuration

We have confirmed that the NETCONF endpoint on XRv is capable of providing full running configuration. Get-config RPC is also capable of getting a subsection of configuration such as:

```
RP/0/0/CPU0:PE2# show run router bgp
```

NETCONF version. using get-config RPC with bgp filter:

```
#295
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="101">
    <get-config>
        <source>
            <running/>
        </source>
        <filter type="subtree">
         <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ipv4-bgp-cfg"/>
        </filter>
    </get-config>
</rpc>
##
```

Which should result in an rpc-reply with similar content:

```
#1133
<?xml version="1.0"?>
<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ipv4-bgp-cfg">
   <instance>
    <instance-name>default</instance-name>
    <instance-as>
     <as>65000</as>
     <four-byte-as>
      <as>10</as>
      <bgp-running></bgp-running>
      <default-vrf>
       <global>
        <global-afs>
         <global-af>
          <af-name>ipv4-unicast</af-name>
          <enable></enable>
          <sourced-networks>
           <sourced-network>
            <network-addr>10.10.10.0</network-addr>
            <network-prefix>24</network-prefix>
           </sourced-network>
          </sourced-networks>
         </global-af>
        </global-afs>
       </global>
       <bgp-entity>
        <neighbors>
         <neighbor>
          <neighbor-address>1.1.1.1</neighbor-address>
          <remote-as>
           <as-xx>0</as-xx>
           <as-yy>44</as-yy>
          </remote-as>
         </neighbor>
        </neighbors>
       </bgp-entity>
      </default-vrf>
     </four-byte-as>
    </instance-as>
   </instance>
  </bgp>

#22
 </data>
</rpc-reply>

##
```

For a comparison, the configuration when viewed from CLI looks like:

```
RP/0/0/CPU0:PE2#sh run router bgp
Sat Mar  3 11:13:43.844 UTC
router bgp 65000.10
 address-family ipv4 unicast
  network 10.10.10.0/24
 !
 neighbor 1.1.1.1
  remote-as 44
 !
!
```

## Summary

As you can see, working with NETCONF endpoint manually is not simple. However sometimes it is necessary to verify NETCONF setup on a device with no tools available. However if you can, use one of existing NETCONF clients e.g:

* [YANG explorer](https://github.com/CiscoDevNet/yang-explorer)
* [Opendaylight SDN controller](http://docs.opendaylight.org/en/stable-nitrogen/user-guide/netconf-user-guide.html)