# ethtool

## man ethtool

```bash
$ man ethtool
ETHTOOL(8)                                                                                 System Manager's Manual                                                                                 ETHTOOL(8)

NAME
       ethtool - query or control network driver and hardware settings

SYNOPSIS
       ethtool devname

       ethtool -h|--help

       ethtool --version

       ethtool -a|--show-pause devname

       ethtool -A|--pause devname [autoneg on|off] [rx on|off] [tx on|off]

       ethtool -c|--show-coalesce devname

       ethtool -C|--coalesce devname [adaptive-rx on|off] [adaptive-tx on|off] [rx-usecs N] [rx-frames N] [rx-usecs-irq N] [rx-frames-irq N] [tx-usecs N] [tx-frames N] [tx-usecs-irq N] [tx-frames-irq N]
              [stats-block-usecs N] [pkt-rate-low N] [rx-usecs-low N] [rx-frames-low N] [tx-usecs-low N] [tx-frames-low N] [pkt-rate-high N] [rx-usecs-high N] [rx-frames-high N] [tx-usecs-high N]
              [tx-frames-high N] [sample-interval N]

       ethtool -g|--show-ring devname

       ethtool -G|--set-ring devname [rx N] [rx-mini N] [rx-jumbo N] [tx N]

       ethtool -i|--driver devname

       ethtool -d|--register-dump devname [raw on|off] [hex on|off] [file name]

       ethtool -e|--eeprom-dump devname [raw on|off] [offset N] [length N]

       ethtool -E|--change-eeprom devname [magic N] [offset N] [length N] [value N]

       ethtool -k|--show-features|--show-offload devname

       ethtool -K|--features|--offload devname feature on|off ...

       ethtool -p|--identify devname [N]

       ethtool -P|--show-permaddr devname

       ethtool -r|--negotiate devname

       ethtool -S|--statistics devname

       ethtool --phy-statistics devname

       ethtool -t|--test devname [offline|online|external_lb]

       ethtool -s devname [speed N] [duplex half|full] [port tp|aui|bnc|mii] [mdix auto|on|off] [autoneg on|off] [advertise N] [phyad N] [xcvr internal|external] [wol p|u|m|b|a|g|s|f|d...]
              [sopass xx:yy:zz:aa:bb:cc] [msglvl N | msglvl type on|off ...]

       ethtool -n|-u|--show-nfc|--show-ntuple devname [ rx-flow-hash tcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6 |
              rule N ]

       ethtool -N|-U|--config-nfc|--config-ntuple devname rx-flow-hash tcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6 m|v|t|s|d|f|n|r... |
              flow-type ether|ip4|tcp4|udp4|sctp4|ah4|esp4|ip6|tcp6|udp6|ah6|esp6|sctp6 [src xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]] [dst xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]] [proto N [m N]]
              [src-ip ip-address [m ip-address]] [dst-ip ip-address [m ip-address]] [tos N [m N]] [tclass N [m N]] [l4proto N [m N]] [src-port N [m N]] [dst-port N [m N]] [spi N [m N]] [l4data N [m N]]
              [vlan-etype N [m N]] [vlan N [m N]] [user-def N [m N]] [dst-mac xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]] [action N] [context N] [loc N] |
              delete N

       ethtool -w|--get-dump devname [data filename]

       ethtool -W|--set-dump devname N

       ethtool -T|--show-time-stamping devname

       ethtool -x|--show-rxfh-indir|--show-rxfh devname

       ethtool -X|--set-rxfh-indir|--rxfh devname [hkey xx:yy:zz:aa:bb:cc:...]  [start N] [ equal N | weight W0 W1 ... | default ] [hfunc FUNC] [context CTX | new] [delete]

       ethtool -f|--flash devname file [N]

       ethtool -l|--show-channels devname

       ethtool -L|--set-channels devname [rx N] [tx N] [other N] [combined N]

       ethtool -m|--dump-module-eeprom|--module-info devname [raw on|off] [hex on|off] [offset N] [length N]

       ethtool --show-priv-flags devname

       ethtool --set-priv-flags devname flag on|off ...

       ethtool --show-eee devname

       ethtool --set-eee devname [eee on|off] [tx-lpi on|off] [tx-timer N] [advertise N]

       ethtool --set-phy-tunable devname [ downshift on|off [count N] ] [ fast-link-down on|off [msecs N] ] [ energy-detect-power-down on|off [msecs N] ]

       ethtool --get-phy-tunable devname [downshift] [fast-link-down] [energy-detect-power-down]

       ethtool --reset devname [flags N] [mgmt] [mgmt-shared] [irq] [irq-shared] [dma] [dma-shared] [filter] [filter-shared] [offload] [offload-shared] [mac] [mac-shared] [phy] [phy-shared] [ram] [ram-
              shared] [ap] [ap-shared] [dedicated] [all]

       ethtool --show-fec devname

       ethtool --set-fec devname encoding auto|off|rs|baser [...]

       ethtool -Q|--per-queue devname [queue_mask %x] sub_command ...
               .

DESCRIPTION
       ethtool is used to query and control network device driver and hardware settings, particularly for wired Ethernet devices.

       devname is the name of the network device on which ethtool should operate.

OPTIONS
       ethtool with a single argument specifying the device name prints current settings of the specified device.

       -h --help
              Shows a short help message.

       --version
              Shows the ethtool version number.

       -a --show-pause
              Queries the specified Ethernet device for pause parameter information.

       -A --pause
              Changes the pause parameters of the specified Ethernet device.

           autoneg on|off
                  Specifies whether pause autonegotiation should be enabled.

           rx on|off
                  Specifies whether RX pause should be enabled.

           tx on|off
                  Specifies whether TX pause should be enabled.

       -c --show-coalesce
              Queries the specified network device for coalescing information.

       -C --coalesce
              Changes the coalescing settings of the specified network device.

       -g --show-ring
              Queries the specified network device for rx/tx ring parameter information.

       -G --set-ring
              Changes the rx/tx ring parameters of the specified network device.

           rx N   Changes the number of ring entries for the Rx ring.

           rx-mini N
                  Changes the number of ring entries for the Rx Mini ring.

           rx-jumbo N
                  Changes the number of ring entries for the Rx Jumbo ring.

           tx N   Changes the number of ring entries for the Tx ring.

       -i --driver
              Queries the specified network device for associated driver information.

       -d --register-dump
              Retrieves and prints a register dump for the specified network device.  The register format for some devices is known and decoded others are printed in hex.  When raw is enabled, then ethtool
              dumps the raw register data to stdout.  If file is specified, then use contents of previous raw register dump, rather than reading from the device.

       -e --eeprom-dump
              Retrieves and prints an EEPROM dump for the specified network device.  When raw is enabled, then it dumps the raw EEPROM data to stdout. The length and offset parameters allow dumping certain
              portions of the EEPROM.  Default is to dump the entire EEPROM.

           raw on|off

           offset N

           length N

       -E --change-eeprom
              If  value  is specified, changes EEPROM byte for the specified network device.  offset and value specify which byte and it's new value. If value is not specified, stdin is read and written to
              the EEPROM. The length and offset parameters allow writing to certain portions of the EEPROM.  Because of the persistent nature of writing to the EEPROM, a device-specific magic key  must  be
              specified to prevent the accidental writing to the EEPROM.

       -k --show-features --show-offload
              Queries the specified network device for the state of protocol offload and other features.

       -K --features --offload
              Changes the offload parameters and other features of the specified network device.  The following feature names are built-in and others may be defined by the kernel.

           rx on|off
                  Specifies whether RX checksumming should be enabled.

           tx on|off
                  Specifies whether TX checksumming should be enabled.

           sg on|off
                  Specifies whether scatter-gather should be enabled.

           tso on|off
                  Specifies whether TCP segmentation offload should be enabled.

           ufo on|off
                  Specifies whether UDP fragmentation offload should be enabled

           gso on|off
                  Specifies whether generic segmentation offload should be enabled

           gro on|off
                  Specifies whether generic receive offload should be enabled

           lro on|off
                  Specifies whether large receive offload should be enabled

           rxvlan on|off
                  Specifies whether RX VLAN acceleration should be enabled

           txvlan on|off
                  Specifies whether TX VLAN acceleration should be enabled

           ntuple on|off
                  Specifies whether Rx ntuple filters and actions should be enabled

           rxhash on|off
                  Specifies whether receive hashing offload should be enabled

       -p --identify
              Initiates adapter-specific action intended to enable an operator to easily identify the adapter by sight.  Typically this involves blinking one or more LEDs on the specific network port.

           [ N]   Length of time to perform phys-id, in seconds.

       -P --show-permaddr
              Queries the specified network device for permanent hardware address.

       -r --negotiate
              Restarts auto-negotiation on the specified Ethernet device, if auto-negotiation is enabled.

       -S --statistics
              Queries the specified network device for NIC- and driver-specific statistics.

       --phy-statistics
              Queries the specified network device for PHY specific statistics.

       -t --test
              Executes adapter selftest on the specified network device. Possible test modes are:

           offline
                  Perform full set of tests, possibly interrupting normal operation during the tests,

           online Perform limited set of tests, not interrupting normal operation,

           external_lb
                  Perform full set of tests, as for offline, and additionally an external-loopback test.

       -s --change
              Allows changing some or all settings of the specified network device.  All following options only apply if -s was specified.

           speed N
                  Set speed in Mb/s.  ethtool with just the device name as an argument will show you the supported device speeds.

           duplex half|full
                  Sets full or half duplex mode.

           port tp|aui|bnc|mii
                  Selects device port.

           mdix auto|on|off
                  Selects MDI-X mode for port. May be used to override the automatic detection feature of most adapters. An argument of auto means automatic detection of MDI status, on forces MDI-X (cross‐
                  over) mode, while off means MDI (straight through) mode.  The driver should guarantee that this command takes effect immediately, and if necessary may reset the link to cause  the  change
                  to take effect.

           autoneg on|off
                  Specifies  whether  autonegotiation  should be enabled. Autonegotiation is enabled by default, but in some network devices may have trouble with it, so you can disable it if really neces‐
                  sary.

           advertise N
                  Sets the speed and duplex advertised by autonegotiation.  The argument is a hexadecimal value using one or a combination of the following values:

                  0x001                  10baseT Half
                  0x002                  10baseT Full
                  0x004                  100baseT Half
                  0x008                  100baseT Full
                  0x80000000000000000    100baseT1 Full
                  0x010                  1000baseT Half               (not supported by IEEE standards)
                  0x020                  1000baseT Full
                  0x100000000000000000   1000baseT1 Full
                  0x20000                1000baseKX Full
                  0x20000000000          1000baseX Full
                  0x800000000000         2500baseT Full
                  0x8000                 2500baseX Full               (not supported by IEEE standards)
                  0x1000000000000        5000baseT Full
                  0x1000                 10000baseT Full
                  0x40000                10000baseKX4 Full
                  0x80000                10000baseKR Full
                  0x100000               10000baseR_FEC
                  0x40000000000          10000baseCR  Full
                  0x80000000000          10000baseSR  Full
                  0x100000000000         10000baseLR  Full
                  0x200000000000         10000baseLRM Full
                  0x400000000000         10000baseER  Full
                  0x200000               20000baseMLD2 Full           (not supported by IEEE standards)
                  0x400000               20000baseKR2 Full            (not supported by IEEE standards)
                  0x80000000             25000baseCR Full
                  0x100000000            25000baseKR Full
                  0x200000000            25000baseSR Full
                  0x800000               40000baseKR4 Full
                  0x1000000              40000baseCR4 Full
                  0x2000000              40000baseSR4 Full
                  0x4000000              40000baseLR4 Full
                  0x400000000            50000baseCR2 Full
                  0x800000000            50000baseKR2 Full
                  0x10000000000          50000baseSR2 Full
                  0x10000000000000       50000baseKR Full
                  0x20000000000000       50000baseSR Full
                  0x40000000000000       50000baseCR Full
                  0x80000000000000       50000baseLR_ER_FR Full
                  0x100000000000000      50000baseDR Full
                  0x8000000              56000baseKR4 Full
                  0x10000000             56000baseCR4 Full
                  0x20000000             56000baseSR4 Full
                  0x40000000             56000baseLR4 Full
                  0x1000000000           100000baseKR4 Full
                  0x2000000000           100000baseSR4 Full
                  0x4000000000           100000baseCR4 Full
                  0x8000000000           100000baseLR4_ER4 Full
                  0x200000000000000      100000baseKR2 Full
                  0x400000000000000      100000baseSR2 Full
                  0x800000000000000      100000baseCR2 Full
                  0x1000000000000000     100000baseLR2_ER2_FR2 Full
                  0x2000000000000000     100000baseDR2 Full
                  0x4000000000000000     200000baseKR4 Full
                  0x8000000000000000     200000baseSR4 Full
                  0x10000000000000000    200000baseLR4_ER4_FR4 Full
                  0x20000000000000000    200000baseDR4 Full
                  0x40000000000000000    200000baseCR4 Full

           phyad N
                  PHY address.

           xcvr internal|external
                  Selects transceiver type. Currently only internal and external can be specified, in the future further types might be added.

           wol p|u|m|b|a|g|s|f|d...
                  Sets Wake-on-LAN options.  Not all devices support this.  The argument to this option is a string of characters specifying which options to enable.

                  p   Wake on PHY activity
                  u   Wake on unicast messages
                  m   Wake on multicast messages
                  b   Wake on broadcast messages
                  a   Wake on ARP
                  g   Wake on MagicPacket™
                  s   Enable SecureOn™ password for MagicPacket™
                  f   Wake on filter(s)
                  d   Disable (wake on nothing).  This option clears all previous options.

           sopass xx:yy:zz:aa:bb:cc
                  Sets the SecureOn™ password.  The argument to this option must be 6 bytes in Ethernet MAC hex format (xx:yy:zz:aa:bb:cc).

           msglvl N
           msglvl type on|off ...
                  Sets the driver message type flags by name or number. type names the type of message to enable or disable; N specifies the new flags numerically. The defined type names and numbers are:

                  drv         0x0001  General driver status
                  probe       0x0002  Hardware probing
                  link        0x0004  Link state
                  timer       0x0008  Periodic status check
                  ifdown      0x0010  Interface being brought down
                  ifup        0x0020  Interface being brought up
                  rx_err      0x0040  Receive error
                  tx_err      0x0080  Transmit error
                  tx_queued   0x0100  Transmit queueing
                  intr        0x0200  Interrupt handling
                  tx_done     0x0400  Transmit completion
                  rx_status   0x0800  Receive completion
                  pktdata     0x1000  Packet contents
                  hw          0x2000  Hardware status
                  wol         0x4000  Wake-on-LAN status

                  The precise meanings of these type flags differ between drivers.

       -n -u --show-nfc --show-ntuple
              Retrieves receive network flow classification options or rules.

           rx-flow-hash tcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6
                  Retrieves the hash options for the specified flow type.

                  tcp4    TCP over IPv4
                  udp4    UDP over IPv4
                  ah4     IPSEC AH over IPv4
                  esp4    IPSEC ESP over IPv4
                  sctp4   SCTP over IPv4
                  tcp6    TCP over IPv6
                  udp6    UDP over IPv6
                  ah6     IPSEC AH over IPv6
                  esp6    IPSEC ESP over IPv6
                  sctp6   SCTP over IPv6

           rule N Retrieves the RX classification rule with the given ID.

       -N -U --config-nfc --config-ntuple
              Configures receive network flow classification options or rules.

           rx-flow-hash tcp4|udp4|ah4|esp4|sctp4|tcp6|udp6|ah6|esp6|sctp6m|v|t|s|d|f|n|r...
                  Configures the hash options for the specified flow type.

                  m   Hash on the Layer 2 destination address of the rx packet.
                  v   Hash on the VLAN tag of the rx packet.
                  t   Hash on the Layer 3 protocol field of the rx packet.
                  s   Hash on the IP source address of the rx packet.
                  d   Hash on the IP destination address of the rx packet.
                  f   Hash on bytes 0 and 1 of the Layer 4 header of the rx packet.
                  n   Hash on bytes 2 and 3 of the Layer 4 header of the rx packet.
                  r   Discard all packets of this flow type. When this option is set,  all
                      other options are ignored.

           flow-type ether|ip4|tcp4|udp4|sctp4|ah4|esp4|ip6|tcp6|udp6|ah6|esp6|sctp6
                  Inserts or updates a classification rule for the specified flow type.

                  ether   Ethernet
                  ip4     Raw IPv4
                  tcp4    TCP over IPv4
                  udp4    UDP over IPv4
                  sctp4   SCTP over IPv4
                  ah4     IPSEC AH over IPv4
                  esp4    IPSEC ESP over IPv4
                  ip6     Raw IPv6
                  tcp6    TCP over IPv6
                  udp6    UDP over IPv6
                  sctp6   SCTP over IPv6
                  ah6     IPSEC AH over IPv6
                  esp6    IPSEC ESP over IPv6

           For  all  fields  that  allow  both a value and a mask to be specified, the mask may be specified immediately after the value using the m keyword, or separately using the field name keyword with
           -mask appended, e.g. src-mask.

           src xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]
                  Includes the source MAC address, specified as 6 bytes in hexadecimal separated by colons, along with an optional mask.  Valid only for flow-type ether.

           dst xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]
                  Includes the destination MAC address, specified as 6 bytes in hexadecimal separated by colons, along with an optional mask.  Valid only for flow-type ether.

           proto N [m N]
                  Includes the Ethernet protocol number (ethertype) and an optional mask.  Valid only for flow-type ether.

           src-ip ip-address [m ip-address]
                  Specify the source IP address of the incoming packet to match along with an optional mask.  Valid for all IP based flow-types.

           dst-ip ip-address [m ip-address]
                  Specify the destination IP address of the incoming packet to match along with an optional mask.  Valid for all IP based flow-types.

           tos N [m N]
                  Specify the value of the Type of Service field in the incoming packet to match along with an optional mask.  Applies to all IPv4 based flow-types.

           tclass N [m N]
                  Specify the value of the Traffic Class field in the incoming packet to match along with an optional mask.  Applies to all IPv6 based flow-types.

           l4proto N [m N]
                  Includes the layer 4 protocol number and optional mask.  Valid only for flow-types ip4 and ip6.

           src-port N [m N]
                  Specify the value of the source port field (applicable to TCP/UDP packets) in the incoming packet to match along with an optional mask.  Valid for flow-types ip4, tcp4,  udp4,  and  sctp4
                  and their IPv6 equivalents.

           dst-port N [m N]
                  Specify  the  value  of  the  destination port field (applicable to TCP/UDP packets)in the incoming packet to match along with an optional mask.  Valid for flow-types ip4, tcp4, udp4, and
                  sctp4 and their IPv6 equivalents.

           spi N [m N]
                  Specify the value of the security parameter index field (applicable to AH/ESP packets)in the incoming packet to match along with an optional mask.  Valid for flow-types ip4, ah4, and esp4
                  and their IPv6 equivalents.

           l4data N [m N]
                  Specify the value of the first 4 Bytes of Layer 4 in the incoming packet to match along with an optional mask.  Valid for ip4 and ip6 flow-types.

           vlan-etype N [m N]
                  Includes the VLAN tag Ethertype and an optional mask.

           vlan N [m N]
                  Includes the VLAN tag and an optional mask.

           user-def N [m N]
                  Includes 64-bits of user-specific data and an optional mask.

           dst-mac xx:yy:zz:aa:bb:cc [m xx:yy:zz:aa:bb:cc]
                  Includes the destination MAC address, specified as 6 bytes in hexadecimal separated by colons, along with an optional mask.  Valid for all IP based flow-types.

           action N
                  Specifies the Rx queue to send packets to, or some other action.

                  -1            Drop the matched flow
                  -2            Use the matched flow as a Wake-on-LAN filter
                  0 or higher   Rx queue to route the flow

           context N
                  Specifies the RSS context to spread packets over multiple queues; either 0 for the default RSS context, or a value returned by ethtool -X ... context new.

           vf N   Specifies the Virtual Function the filter applies to. Not compatible with action.

           queue N
                  Specifies the Rx queue to send packets to. Not compatible with action.

           loc N  Specify the location/ID to insert the rule. This will overwrite any rule present in that location and will not go through any of the rule ordering process.

           delete N
                  Deletes the RX classification rule with the given ID.

       -w --get-dump
              Retrieves  and  prints  firmware  dump  for  the specified network device.  By default, it prints out the dump flag, version and length of the dump data.  When data is indicated, then ethtool
              fetches the dump data and directs it to a file.

       -W --set-dump
              Sets the dump flag for the device.

       -T --show-time-stamping
              Show the device's time stamping capabilities and associated PTP hardware clock.

       -x --show-rxfh-indir --show-rxfh
              Retrieves the receive flow hash indirection table and/or RSS hash key.

       -X --set-rxfh-indir --rxfh
              Configures the receive flow hash indirection table and/or RSS hash key.

           hkey   Sets RSS hash key of the specified network device. RSS hash key should be of device supported length.  Hash key format must be in xx:yy:zz:aa:bb:cc format meaning both the  nibbles  of  a
                  byte should be mentioned even if a nibble is zero.

           hfunc  Sets RSS hash function of the specified network device.  List of RSS hash functions which kernel supports is shown as a part of the --show-rxfh command output.

           start N
                  For the equal and weight options, sets the starting receive queue for spreading flows to N.

           equal N
                  Sets the receive flow hash indirection table to spread flows evenly between the first N receive queues.

           weight W0 W1 ...
                  Sets  the  receive flow hash indirection table to spread flows between receive queues according to the given weights.  The sum of the weights must be non-zero and must not exceed the size
                  of the indirection table.

           default
                  Sets the receive flow hash indirection table to its default value.

           context CTX | new
                  Specifies an RSS context to act on; either new to allocate a new RSS context, or CTX, a value returned by a previous ... context new.

           delete Delete the specified RSS context.  May only be used in conjunction with context and a non-zero CTX value.

       -f --flash
              Write a firmware image to flash or other non-volatile memory on the device.

           file   Specifies the filename of the firmware image.  The firmware must first be installed in one of the directories where the kernel firmware  loader  or  firmware  agent  will  look,  such  as
                  /lib/firmware.

           N      If  the device stores multiple firmware images in separate regions of non-volatile memory, this parameter may be used to specify which region is to be written.  The default is 0, request‐
                  ing that all regions are written.  All other values are driver-dependent.

       -l --show-channels
              Queries the specified network device for the numbers of channels it has.  A channel is an IRQ and the set of queues that can trigger that IRQ.

       -L --set-channels
              Changes the numbers of channels of the specified network device.

           rx N   Changes the number of channels with only receive queues.

           tx N   Changes the number of channels with only transmit queues.

           other N
                  Changes the number of channels used only for other purposes e.g. link interrupts or SR-IOV co-ordination.

           combined N
                  Changes the number of multi-purpose channels.

       -m --dump-module-eeprom --module-info
              Retrieves and if possible decodes the EEPROM from plugin modules, e.g SFP+, QSFP.  If the driver and module support it, the optical diagnostic information is also read and decoded.

       --show-priv-flags
              Queries the specified network device for its private flags.  The names and meanings of private flags (if any) are defined by each network device driver.

       --set-priv-flags
              Sets the device's private flags as specified.

           flag on|off Sets the state of the named private flag.

       --show-eee
              Queries the specified network device for its support of Energy-Efficient Ethernet (according to the IEEE 802.3az specifications)

       --set-eee
              Sets the device EEE behaviour.

           eee on|off
                  Enables/disables the device support of EEE.

           tx-lpi on|off
                  Determines whether the device should assert its Tx LPI.

           advertise N
                  Sets the speeds for which the device should advertise EEE capabilities.  Values are as for --change advertise

           tx-timer N
                  Sets the amount of time the device should stay in idle mode prior to asserting its Tx LPI (in microseconds). This has meaning only when Tx LPI is enabled.

       --set-phy-tunable
              Sets the PHY tunable parameters.

           downshift on|off
                  Specifies whether downshift should be enabled.

                  count N
                      Sets the PHY downshift re-tries count.

           fast-link-down on|off
                  Specifies whether Fast Link Down should be enabled and time until link down (if supported).

                  msecs N
                      Sets the period after which the link is reported as down. Note that the PHY may choose
                      the closest supported value. Only on reading back the tunable do you get the actual value.

           energy-detect-power-down on|off
                  Specifies whether Energy Detect Power Down (EDPD) should be enabled (if supported).  This will put the RX and TX circuit blocks into a low power mode, and the PHY will wake up periodical‐
                  ly to send link pulses to avoid any lock-up situation with a peer PHY that may also have EDPD enabled. By default, this setting will also enable the periodic transmission of TX pulses.

                  msecs N
                      Some PHYs support configuration of the wake-up interval to send TX pulses.
                      This setting allows the control of this interval, and 0 disables TX pulses
                      if the PHY supports this. Disabling TX pulses can create a lock-up situation
                      where neither of the PHYs wakes the other one. If unspecified the default
                      value (in milliseconds) will be used by the PHY.

       --get-phy-tunable
              Gets the PHY tunable parameters.

           downshift
                  For  operation  in  cabling  environments  that  are  incompatible  with  1000BASE-T, PHY device provides an automatic link speed downshift operation.  Link speed downshift after N failed
                  1000BASE-T auto-negotiation attempts.  Downshift is useful where cable does not have the 4 pairs instance.

                  Gets the PHY downshift count/status.

           fast-link-down
                  Depending on the mode it may take 0.5s - 1s until a broken link is reported as down.  In certain use cases a link-down event needs to be reported as soon as possible.  Some PHYs support a
                  Fast Link Down Feature and may allow configuration of the delay before a broken link is reported as being down.

                  Gets the PHY Fast Link Down status / period.

           energy-detect-power-down
                  Gets the current configured setting for Energy Detect Power Down (if supported).

       --reset
              Reset hardware components specified by flags and components listed below

           flags N
                  Resets the components based on direct flags mask

           mgmt   Management processor

           irq    Interrupt requester

           dma    DMA engine

           filter Filtering/flow direction

           offload
                  Protocol offload

           mac    Media access controller

           phy    Transceiver/PHY

           ram    RAM shared between multiple components ap Application Processor

           dedicated
                  All components dedicated to this interface

           all    All components used by this interface, even if shared

       --show-fec
              Queries the specified network device for its support of Forward Error Correction.

       --set-fec
              Configures Forward Error Correction for the specified network device.

              Forward  Error  Correction  modes  selected  by  a user are expected to be persisted after any hotplug events. If a module is swapped that does not support the current FEC mode, the driver or
              firmware must take the link down administratively and report the problem in the system logs for users to correct.

           encoding auto|off|rs|baser [...]

                  Sets the FEC encoding for the device.  Combinations of options are specified as e.g.  encoding auto rs ; the semantics of such combinations vary between drivers.

                  auto    Use the driver's default encoding
                  off     Turn off FEC
                  RS      Force RS-FEC encoding
                  BaseR   Force BaseR encoding

       -Q|--per-queue
              Applies provided sub command to specific queues.

           queue_mask %x
                  Sets the specific queues which the sub command is applied to.  If queue_mask is not set, the sub command will be applied to all queues.

           sub_command
                  Sub command to apply. The supported sub commands include --show-coalesce and --coalesce.

BUGS
       Not supported (in part or whole) on all network drivers.

AUTHOR
       ethtool was written by David Miller.

       Modifications by Jeff Garzik, Tim Hockin, Jakub Jelinek, Andre Majorel, Eli Kupermann, Scott Feldman, Andi Kleen, Alexander Duyck, Sucheta Chakraborty, Jesse Brandeburg, Ben Hutchings,  Scott  Bran‐
       den.

AVAILABILITY
       ethtool is available from ⟨http://www.kernel.org/pub/software/network/ethtool/⟩

Ethtool version 5.4                                                                              January 2020                                                                                      ETHTOOL(8)
```



## 查看网口基本信息

```bash
$ ethtool ens33
Settings for ens33:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Supported pause frame use: No
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Advertised pause frame use: No
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Speed: 1000Mb/s
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off (auto)
Cannot get wake-on-lan settings: Operation not permitted
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes
```

## 查询网口驱动相关信息

```bash
$ ethtool -i ens33
driver: e1000
version: 5.10.0
firmware-version: 
expansion-rom-version: 
bus-info: 0000:02:01.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no
```

## 查询网口收发包统计信息

```bash
$ ethtool -S ens33
NIC statistics:
     rx_packets: 278
     tx_packets: 245
     rx_bytes: 200034
     tx_bytes: 47526
     rx_broadcast: 0
     tx_broadcast: 0
     rx_multicast: 0
     tx_multicast: 0
     rx_errors: 0
     tx_errors: 0
     tx_dropped: 0
     multicast: 0
     collisions: 0
     rx_length_errors: 0
     rx_over_errors: 0
     rx_crc_errors: 0
     rx_frame_errors: 0
     rx_no_buffer_count: 0
     rx_missed_errors: 0
     tx_aborted_errors: 0
     tx_carrier_errors: 0
     tx_fifo_errors: 0
     tx_heartbeat_errors: 0
     tx_window_errors: 0
     tx_abort_late_coll: 0
     tx_deferred_ok: 0
     tx_single_coll_ok: 0
     tx_multi_coll_ok: 0
     tx_timeout_count: 0
     tx_restart_queue: 0
     rx_long_length_errors: 0
     rx_short_length_errors: 0
     rx_align_errors: 0
     tx_tcp_seg_good: 5
     tx_tcp_seg_failed: 0
     rx_flow_control_xon: 0
     rx_flow_control_xoff: 0
     tx_flow_control_xon: 0
     tx_flow_control_xoff: 0
     rx_long_byte_count: 200034
     rx_csum_offload_good: 266
     rx_csum_offload_errors: 0
     alloc_rx_buff_failed: 0
     tx_smbus: 0
     rx_smbus: 0
     dropped_smbus: 0
```

## 显示网卡 offload 参数的状态

```bash
$ ethtool -k ens33 
Features for ens33:
rx-checksumming: off
tx-checksumming: on
	tx-checksum-ipv4: off [fixed]
	tx-checksum-ip-generic: on
	tx-checksum-ipv6: off [fixed]
	tx-checksum-fcoe-crc: off [fixed]
	tx-checksum-sctp: off [fixed]
scatter-gather: on
	tx-scatter-gather: on
	tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: off [fixed]
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: off [fixed]
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
tx-gso-list: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off
rx-all: off
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: off [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
```

### 配置网卡 offload 参数

```bash
$ ethtool -K eth0 rx-checksum on|off
$ ethtool -K eth0 tx-checksum-ip-generic on|off
$ ethtool -K eth0 tso on|off
$ ethtool -K eth0 ufo on | off
$ ethtool -K eth0 gso on | off
$ ethtool -K eth0 ntuple on | off
```



## 查看多队列网卡的队列使用情况

```bash
$ ethtool -l eth0
```

### 修改队列数

```bash
$ ethtool -L eth0 combined 32
```

### 查看网卡队列绑定信息

```bash
# 注意修改网卡名称eth0
for i in `cat /proc/interrupts | grep eth0 \
 | awk -F: '{print $1}'`; do echo $i && cat /proc/irq/$i/smp_affinity ; done
```

### 在文件系统查看生效的网卡队列数量

```bash
$ ls /sys/class/net/eth0/queues
```



















# 参考

[Linux系统下查看网卡相关数据_ethtool bus-info_香烟头的博客-CSDN博客](https://blog.csdn.net/yunlianglinfeng/article/details/120458288)