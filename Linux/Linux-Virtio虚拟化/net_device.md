## net_device

网络设备结构体，用于描述一个网络设备

```c
/include/netdevice.h
/**
 *	struct net_device - The DEVICE structure.
 *
 *	Actually, this whole structure is a big mistake.  It mixes I/O
 *	data with strictly "high-level" data, and it has to know about
 *	almost every data structure used in the INET module.
 *
 *	@name:	This is the first field of the "visible" part of this structure
 *		(i.e. as seen by users in the "Space.c" file).  It is the name
 *		of the interface.
 *
 *	@name_node:	Name hashlist node
 *	@ifalias:	SNMP alias
 *	@mem_end:	Shared memory end
 *	@mem_start:	Shared memory start
 *	@base_addr:	Device I/O address
 *	@irq:		Device IRQ number
 *
 *	@state:		Generic network queuing layer state, see netdev_state_t
 *	@dev_list:	The global list of network devices
 *	@napi_list:	List entry used for polling NAPI devices
 *	@unreg_list:	List entry  when we are unregistering the
 *			device; see the function unregister_netdev
 *	@close_list:	List entry used when we are closing the device
 *	@ptype_all:     Device-specific packet handlers for all protocols
 *	@ptype_specific: Device-specific, protocol-specific packet handlers
 *
 *	@adj_list:	Directly linked devices, like slaves for bonding
 *	@features:	配置硬件功能特性
 *	@hw_features:	User-changeable features
 *
 *	@wanted_features:	User-requested features
 *	@vlan_features:		Mask of features inheritable by VLAN devices
 *
 *	@hw_enc_features:	Mask of features inherited by encapsulating devices
 *				This field indicates what encapsulation
 *				offloads the hardware is capable of doing,
 *				and drivers will need to set them appropriately.
 *
 *	@mpls_features:	Mask of features inheritable by MPLS
 *	@gso_partial_features: value(s) from NETIF_F_GSO\*
 *
 *	@ifindex:	interface index
 *	@group:		The group the device belongs to
 *
 *	@stats:		Statistics struct, which was left as a legacy, use
 *			rtnl_link_stats64 instead
 *
 *	@rx_dropped:	Dropped packets by core network,
 *			do not use this in drivers
 *	@tx_dropped:	Dropped packets by core network,
 *			do not use this in drivers
 *	@rx_nohandler:	nohandler dropped packets by core network on
 *			inactive devices, do not use this in drivers
 *	@carrier_up_count:	Number of times the carrier has been up
 *	@carrier_down_count:	Number of times the carrier has been down
 *
 *	@wireless_handlers:	List of functions to handle Wireless Extensions,
 *				instead of ioctl,
 *				see <net/iw_handler.h> for details.
 *	@wireless_data:	Instance data managed by the core of wireless extensions
 *
 *	@netdev_ops:	Includes several pointers to callbacks,
 *			if one wants to override the ndo_*() functions
 *	@ethtool_ops:	Management operations
 *	@l3mdev_ops:	Layer 3 master device operations
 *	@ndisc_ops:	Includes callbacks for different IPv6 neighbour
 *			discovery handling. Necessary for e.g. 6LoWPAN.
 *	@xfrmdev_ops:	Transformation offload operations
 *	@tlsdev_ops:	Transport Layer Security offload operations
 *	@header_ops:	Includes callbacks for creating,parsing,caching,etc
 *			of Layer 2 headers.
 *
 *	@flags:		Interface flags (a la BSD)
 *	@priv_flags:	Like 'flags' but invisible to userspace,
 *			see if.h for the definitions
 *	@gflags:	Global flags ( kept as legacy )
 *	@padded:	How much padding added by alloc_netdev()
 *	@operstate:	RFC2863 operstate
 *	@link_mode:	Mapping policy to operstate
 *	@if_port:	Selectable AUI, TP, ...
 *	@dma:		DMA channel
 *	@mtu:		Interface MTU value
 *	@min_mtu:	Interface Minimum MTU value
 *	@max_mtu:	Interface Maximum MTU value
 *	@type:		Interface hardware type
 *	@hard_header_len: Maximum hardware header length.
 *	@min_header_len:  Minimum hardware header length
 *
 *	@needed_headroom: Extra headroom the hardware may need, but not in all
 *			  cases can this be guaranteed
 *	@needed_tailroom: Extra tailroom the hardware may need, but not in all
 *			  cases can this be guaranteed. Some cases also use
 *			  LL_MAX_HEADER instead to allocate the skb
 *
 *	interface address info:
 *
 * 	@perm_addr:		Permanent hw address
 * 	@addr_assign_type:	Hw address assignment type
 * 	@addr_len:		Hardware address length
 *	@upper_level:		Maximum depth level of upper devices.
 *	@lower_level:		Maximum depth level of lower devices.
 *	@neigh_priv_len:	Used in neigh_alloc()
 * 	@dev_id:		Used to differentiate devices that share
 * 				the same link layer address
 * 	@dev_port:		Used to differentiate devices that share
 * 				the same function
 *	@addr_list_lock:	XXX: need comments on this one
 *	@name_assign_type:	network interface name assignment type
 *	@uc_promisc:		Counter that indicates promiscuous mode
 *				has been enabled due to the need to listen to
 *				additional unicast addresses in a device that
 *				does not implement ndo_set_rx_mode()
 *	@uc:			unicast mac addresses
 *	@mc:			multicast mac addresses
 *	@dev_addrs:		list of device hw addresses
 *	@queues_kset:		Group of all Kobjects in the Tx and RX queues
 *	@promiscuity:		Number of times the NIC is told to work in
 *				promiscuous mode; if it becomes 0 the NIC will
 *				exit promiscuous mode
 *	@allmulti:		Counter, enables or disables allmulticast mode
 *
 *	@vlan_info:	VLAN info
 *	@dsa_ptr:	dsa specific data
 *	@tipc_ptr:	TIPC specific data
 *	@atalk_ptr:	AppleTalk link
 *	@ip_ptr:	IPv4 specific data
 *	@dn_ptr:	DECnet specific data
 *	@ip6_ptr:	IPv6 specific data
 *	@ax25_ptr:	AX.25 specific data
 *	@ieee80211_ptr:	IEEE 802.11 specific data, assign before registering
 *	@ieee802154_ptr: IEEE 802.15.4 low-rate Wireless Personal Area Network
 *			 device struct
 *	@mpls_ptr:	mpls_dev struct pointer
 *	@mctp_ptr:	MCTP specific data
 *
 *	@dev_addr:	Hw address (before bcast,
 *			because most packets are unicast)
 *
 *	@_rx:			Array of RX queues
 *	@num_rx_queues:		Number of RX queues
 *				allocated at register_netdev() time
 *	@real_num_rx_queues: 	Number of RX queues currently active in device
 *	@xdp_prog:		XDP sockets filter program pointer
 *	@gro_flush_timeout:	timeout for GRO layer in NAPI
 *	@napi_defer_hard_irqs:	If not zero, provides a counter that would
 *				allow to avoid NIC hard IRQ, on busy queues.
 *
 *	@rx_handler:		handler for received packets
 *	@rx_handler_data: 	XXX: need comments on this one
 *	@miniq_ingress:		ingress/clsact qdisc specific data for
 *				ingress processing
 *	@ingress_queue:		XXX: need comments on this one
 *	@nf_hooks_ingress:	netfilter hooks executed for ingress packets
 *	@broadcast:		hw bcast address
 *
 *	@rx_cpu_rmap:	CPU reverse-mapping for RX completion interrupts,
 *			indexed by RX queue number. Assigned by driver.
 *			This must only be set if the ndo_rx_flow_steer
 *			operation is defined
 *	@index_hlist:		Device index hash chain
 *
 *	@_tx:			Array of TX queues
 *	@num_tx_queues:		Number of TX queues allocated at alloc_netdev_mq() time
 *	@real_num_tx_queues: 	Number of TX queues currently active in device
 *	@qdisc:			Root qdisc from userspace point of view
 *	@tx_queue_len:		Max frames per queue allowed
 *	@tx_global_lock: 	XXX: need comments on this one
 *	@xdp_bulkq:		XDP device bulk queue
 *	@xps_maps:		all CPUs/RXQs maps for XPS device
 *
 *	@xps_maps:	XXX: need comments on this one
 *	@miniq_egress:		clsact qdisc specific data for
 *				egress processing
 *	@qdisc_hash:		qdisc hash table
 *	@watchdog_timeo:	Represents the timeout that is used by
 *				the watchdog (see dev_watchdog())
 *	@watchdog_timer:	List of timers
 *
 *	@proto_down_reason:	reason a netdev interface is held down
 *	@pcpu_refcnt:		Number of references to this device
 *	@dev_refcnt:		Number of references to this device
 *	@todo_list:		Delayed register/unregister
 *	@link_watch_list:	XXX: need comments on this one
 *
 *	@reg_state:		Register/unregister state machine
 *	@dismantle:		Device is going to be freed
 *	@rtnl_link_state:	This enum represents the phases of creating
 *				a new link
 *
 *	@needs_free_netdev:	Should unregister perform free_netdev?
 *	@priv_destructor:	Called from unregister
 *	@npinfo:		XXX: need comments on this one
 * 	@nd_net:		Network namespace this network device is inside
 *
 * 	@ml_priv:	Mid-layer private
 *	@ml_priv_type:  Mid-layer private type
 * 	@lstats:	Loopback statistics
 * 	@tstats:	Tunnel statistics
 * 	@dstats:	Dummy statistics
 * 	@vstats:	Virtual ethernet statistics
 *
 *	@garp_port:	GARP
 *	@mrp_port:	MRP
 *
 *	@dev:		Class/net/name entry
 *	@sysfs_groups:	Space for optional device, statistics and wireless
 *			sysfs groups
 *
 *	@sysfs_rx_queue_group:	Space for optional per-rx queue attributes
 *	@rtnl_link_ops:	Rtnl_link_ops
 *
 *	@gso_max_size:	Maximum size of generic segmentation offload
 *	@gso_max_segs:	Maximum number of segments that can be passed to the
 *			NIC for GSO
 *
 *	@dcbnl_ops:	Data Center Bridging netlink ops
 *	@num_tc:	Number of traffic classes in the net device
 *	@tc_to_txq:	XXX: need comments on this one
 *	@prio_tc_map:	XXX: need comments on this one
 *
 *	@fcoe_ddp_xid:	Max exchange id for FCoE LRO by ddp
 *
 *	@priomap:	XXX: need comments on this one
 *	@phydev:	Physical device may attach itself
 *			for hardware timestamping
 *	@sfp_bus:	attached &struct sfp_bus structure.
 *
 *	@qdisc_tx_busylock: lockdep class annotating Qdisc->busylock spinlock
 *	@qdisc_running_key: lockdep class annotating Qdisc->running seqcount
 *
 *	@proto_down:	protocol port state information can be sent to the
 *			switch driver and used to set the phys state of the
 *			switch port.
 *
 *	@wol_enabled:	Wake-on-LAN is enabled
 *
 *	@threaded:	napi threaded mode is enabled
 *
 *	@net_notifier_list:	List of per-net netdev notifier block
 *				that follow this device when it is moved
 *				to another network namespace.
 *
 *	@macsec_ops:    MACsec offloading ops
 *
 *	@udp_tunnel_nic_info:	static structure describing the UDP tunnel
 *				offload capabilities of the device
 *	@udp_tunnel_nic:	UDP tunnel offload state
 *	@xdp_state:		stores info on attached XDP BPF programs
 *
 *	@nested_level:	Used as as a parameter of spin_lock_nested() of
 *			dev->addr_list_lock.
 *	@unlink_list:	As netif_addr_lock() can be called recursively,
 *			keep a list of interfaces to be deleted.
 *
 *	FIXME: cleanup struct net_device such that network protocol info
 *	moves out.
 */

struct net_device {
	char			name[IFNAMSIZ];
	struct netdev_name_node	*name_node;
	struct dev_ifalias	__rcu *ifalias;
	/*
	 *	I/O specific fields
	 *	FIXME: Merge these and struct ifmap into one
	 */
	unsigned long		mem_end;
	unsigned long		mem_start;
	unsigned long		base_addr;

	/*
	 *	Some hardware also needs these fields (state,dev_list,
	 *	napi_list,unreg_list,close_list) but they are not
	 *	part of the usual set specified in Space.c.
	 */

	unsigned long		state;

	struct list_head	dev_list;
	struct list_head	napi_list;
	struct list_head	unreg_list;
	struct list_head	close_list;
	struct list_head	ptype_all;
	struct list_head	ptype_specific;

	struct {
		struct list_head upper;
		struct list_head lower;
	} adj_list;

	/* Read-mostly cache-line for fast-path access */
	unsigned int		flags;
	unsigned int		priv_flags;
	const struct net_device_ops *netdev_ops;
	int			ifindex;
	unsigned short		gflags;
	unsigned short		hard_header_len;

	/* Note : dev->mtu is often read without holding a lock.
	 * Writers usually hold RTNL.
	 * It is recommended to use READ_ONCE() to annotate the reads,
	 * and to use WRITE_ONCE() to annotate the writes.
	 */
	unsigned int		mtu;
	unsigned short		needed_headroom;
	unsigned short		needed_tailroom;

	netdev_features_t	features;
	netdev_features_t	hw_features;
	netdev_features_t	wanted_features;
	netdev_features_t	vlan_features;
	netdev_features_t	hw_enc_features;
	netdev_features_t	mpls_features;
	netdev_features_t	gso_partial_features;

	unsigned int		min_mtu;
	unsigned int		max_mtu;
	unsigned short		type;
	unsigned char		min_header_len;
	unsigned char		name_assign_type;

	int			group;

	struct net_device_stats	stats; /* not used by modern drivers */

	atomic_long_t		rx_dropped;
	atomic_long_t		tx_dropped;
	atomic_long_t		rx_nohandler;

	/* Stats to monitor link on/off, flapping */
	atomic_t		carrier_up_count;
	atomic_t		carrier_down_count;

#ifdef CONFIG_WIRELESS_EXT
	const struct iw_handler_def *wireless_handlers;
	struct iw_public_data	*wireless_data;
#endif
	const struct ethtool_ops *ethtool_ops;
#ifdef CONFIG_NET_L3_MASTER_DEV
	const struct l3mdev_ops	*l3mdev_ops;
#endif
#if IS_ENABLED(CONFIG_IPV6)
	const struct ndisc_ops *ndisc_ops;
#endif

#ifdef CONFIG_XFRM_OFFLOAD
	const struct xfrmdev_ops *xfrmdev_ops;
#endif

#if IS_ENABLED(CONFIG_TLS_DEVICE)
	const struct tlsdev_ops *tlsdev_ops;
#endif

	const struct header_ops *header_ops;

	unsigned char		operstate;
	unsigned char		link_mode;

	unsigned char		if_port;
	unsigned char		dma;

	/* Interface address info. */
	unsigned char		perm_addr[MAX_ADDR_LEN];
	unsigned char		addr_assign_type;
	unsigned char		addr_len;
	unsigned char		upper_level;
	unsigned char		lower_level;

	unsigned short		neigh_priv_len;
	unsigned short          dev_id;
	unsigned short          dev_port;
	unsigned short		padded;

	spinlock_t		addr_list_lock;
	int			irq;

	struct netdev_hw_addr_list	uc;
	struct netdev_hw_addr_list	mc;
	struct netdev_hw_addr_list	dev_addrs;

#ifdef CONFIG_SYSFS
	struct kset		*queues_kset;
#endif
#ifdef CONFIG_LOCKDEP
	struct list_head	unlink_list;
#endif
	unsigned int		promiscuity;
	unsigned int		allmulti;
	bool			uc_promisc;
#ifdef CONFIG_LOCKDEP
	unsigned char		nested_level;
#endif


	/* Protocol-specific pointers */

#if IS_ENABLED(CONFIG_VLAN_8021Q)
	struct vlan_info __rcu	*vlan_info;
#endif
#if IS_ENABLED(CONFIG_NET_DSA)
	struct dsa_port		*dsa_ptr;
#endif
#if IS_ENABLED(CONFIG_TIPC)
	struct tipc_bearer __rcu *tipc_ptr;
#endif
#if IS_ENABLED(CONFIG_IRDA) || IS_ENABLED(CONFIG_ATALK)
	void 			*atalk_ptr;
#endif
	struct in_device __rcu	*ip_ptr;
#if IS_ENABLED(CONFIG_DECNET)
	struct dn_dev __rcu     *dn_ptr;
#endif
	struct inet6_dev __rcu	*ip6_ptr;
#if IS_ENABLED(CONFIG_AX25)
	void			*ax25_ptr;
#endif
	struct wireless_dev	*ieee80211_ptr;
	struct wpan_dev		*ieee802154_ptr;
#if IS_ENABLED(CONFIG_MPLS_ROUTING)
	struct mpls_dev __rcu	*mpls_ptr;
#endif
#if IS_ENABLED(CONFIG_MCTP)
	struct mctp_dev __rcu	*mctp_ptr;
#endif

/*
 * Cache lines mostly used on receive path (including eth_type_trans())
 */
	/* Interface address info used in eth_type_trans() */
	unsigned char		*dev_addr;

	struct netdev_rx_queue	*_rx;
	unsigned int		num_rx_queues;
	unsigned int		real_num_rx_queues;

	struct bpf_prog __rcu	*xdp_prog;
	unsigned long		gro_flush_timeout;
	int			napi_defer_hard_irqs;
	rx_handler_func_t __rcu	*rx_handler;
	void __rcu		*rx_handler_data;

#ifdef CONFIG_NET_CLS_ACT
	struct mini_Qdisc __rcu	*miniq_ingress;
#endif
	struct netdev_queue __rcu *ingress_queue;
#ifdef CONFIG_NETFILTER_INGRESS
	struct nf_hook_entries __rcu *nf_hooks_ingress;
#endif

	unsigned char		broadcast[MAX_ADDR_LEN];
#ifdef CONFIG_RFS_ACCEL
	struct cpu_rmap		*rx_cpu_rmap;
#endif
	struct hlist_node	index_hlist;

/*
 * Cache lines mostly used on transmit path
 */
	struct netdev_queue	*_tx ____cacheline_aligned_in_smp;
	unsigned int		num_tx_queues;
	unsigned int		real_num_tx_queues;
	struct Qdisc		*qdisc;
	unsigned int		tx_queue_len;
	spinlock_t		tx_global_lock;

	struct xdp_dev_bulk_queue __percpu *xdp_bulkq;

#ifdef CONFIG_XPS
	struct xps_dev_maps __rcu *xps_maps[XPS_MAPS_MAX];
#endif
#ifdef CONFIG_NET_CLS_ACT
	struct mini_Qdisc __rcu	*miniq_egress;
#endif

#ifdef CONFIG_NET_SCHED
	DECLARE_HASHTABLE	(qdisc_hash, 4);
#endif
	/* These may be needed for future network-power-down code. */
	struct timer_list	watchdog_timer;
	int			watchdog_timeo;

	u32                     proto_down_reason;

	struct list_head	todo_list;

#ifdef CONFIG_PCPU_DEV_REFCNT
	int __percpu		*pcpu_refcnt;
#else
	refcount_t		dev_refcnt;
#endif

	struct list_head	link_watch_list;

	enum { NETREG_UNINITIALIZED=0,
	       NETREG_REGISTERED,	/* completed register_netdevice */
	       NETREG_UNREGISTERING,	/* called unregister_netdevice */
	       NETREG_UNREGISTERED,	/* completed unregister todo */
	       NETREG_RELEASED,		/* called free_netdev */
	       NETREG_DUMMY,		/* dummy device for NAPI poll */
	} reg_state:8;

	bool dismantle;

	enum {
		RTNL_LINK_INITIALIZED,
		RTNL_LINK_INITIALIZING,
	} rtnl_link_state:16;

	bool needs_free_netdev;
	void (*priv_destructor)(struct net_device *dev);

#ifdef CONFIG_NETPOLL
	struct netpoll_info __rcu	*npinfo;
#endif

	possible_net_t			nd_net;

	/* mid-layer private */
	void				*ml_priv;
	enum netdev_ml_priv_type	ml_priv_type;

	union {
		struct pcpu_lstats __percpu		*lstats;
		struct pcpu_sw_netstats __percpu	*tstats;
		struct pcpu_dstats __percpu		*dstats;
	};

#if IS_ENABLED(CONFIG_GARP)
	struct garp_port __rcu	*garp_port;
#endif
#if IS_ENABLED(CONFIG_MRP)
	struct mrp_port __rcu	*mrp_port;
#endif

	struct device		dev;
	const struct attribute_group *sysfs_groups[4];
	const struct attribute_group *sysfs_rx_queue_group;

	const struct rtnl_link_ops *rtnl_link_ops;

	/* for setting kernel sock attribute on TCP connection setup */
#define GSO_MAX_SIZE		65536
	unsigned int		gso_max_size;
#define GSO_MAX_SEGS		65535
	u16			gso_max_segs;

#ifdef CONFIG_DCB
	const struct dcbnl_rtnl_ops *dcbnl_ops;
#endif
	s16			num_tc;
	struct netdev_tc_txq	tc_to_txq[TC_MAX_QUEUE];
	u8			prio_tc_map[TC_BITMASK + 1];

#if IS_ENABLED(CONFIG_FCOE)
	unsigned int		fcoe_ddp_xid;
#endif
#if IS_ENABLED(CONFIG_CGROUP_NET_PRIO)
	struct netprio_map __rcu *priomap;
#endif
	struct phy_device	*phydev;
	struct sfp_bus		*sfp_bus;
	struct lock_class_key	*qdisc_tx_busylock;
	struct lock_class_key	*qdisc_running_key;
	bool			proto_down;
	unsigned		wol_enabled:1;
	unsigned		threaded:1;

	struct list_head	net_notifier_list;

#if IS_ENABLED(CONFIG_MACSEC)
	/* MACsec management functions */
	const struct macsec_ops *macsec_ops;
#endif
	const struct udp_tunnel_nic_info	*udp_tunnel_nic_info;
	struct udp_tunnel_nic	*udp_tunnel_nic;

	/* protected by rtnl_lock */
	struct bpf_xdp_entity	xdp_state[__MAX_XDP_MODE];
};
```

几个常用的成员

* name

  是网络设备的名称, 网络设备被载入后会出现在ifconfig中, 比如默认的eth0就是这个



### netdev_ops

`net_dev_ops`结构定义了网络设备的钩子函数。

````c
/include/netdevice.h
/*
 * This structure defines the management hooks for network devices.
 * The following hooks can be defined; unless noted otherwise, they are
 * optional and can be filled with a null pointer.
 *
 * int (*ndo_init)(struct net_device *dev);
 *     This function is called once when a network device is registered.
 *     The network device can use this for any late stage initialization
 *     or semantic validation. It can fail with an error code which will
 *     be propagated back to register_netdev.
 *
 * void (*ndo_uninit)(struct net_device *dev);
 *     This function is called when device is unregistered or when registration
 *     fails. It is not called if init fails.
 *
 * int (*ndo_open)(struct net_device *dev);
 *     This function is called when a network device transitions to the up
 *     state.
 *
 * int (*ndo_stop)(struct net_device *dev);
 *     This function is called when a network device transitions to the down
 *     state.
 *
 * netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
 *                               struct net_device *dev);
 *	Called when a packet needs to be transmitted.
 *	Returns NETDEV_TX_OK.  Can return NETDEV_TX_BUSY, but you should stop
 *	the queue before that can happen; it's for obsolete devices and weird
 *	corner cases, but the stack really does a non-trivial amount
 *	of useless work if you return NETDEV_TX_BUSY.
 *	Required; cannot be NULL.
 *
 * netdev_features_t (*ndo_features_check)(struct sk_buff *skb,
 *					   struct net_device *dev
 *					   netdev_features_t features);
 *	Called by core transmit path to determine if device is capable of
 *	performing offload operations on a given packet. This is to give
 *	the device an opportunity to implement any restrictions that cannot
 *	be otherwise expressed by feature flags. The check is called with
 *	the set of features that the stack has calculated and it returns
 *	those the driver believes to be appropriate.
 *
 * u16 (*ndo_select_queue)(struct net_device *dev, struct sk_buff *skb,
 *                         struct net_device *sb_dev);
 *	Called to decide which queue to use when device supports multiple
 *	transmit queues.
 *
 * void (*ndo_change_rx_flags)(struct net_device *dev, int flags);
 *	This function is called to allow device receiver to make
 *	changes to configuration when multicast or promiscuous is enabled.
 *
 * void (*ndo_set_rx_mode)(struct net_device *dev);
 *	This function is called device changes address list filtering.
 *	If driver handles unicast address filtering, it should set
 *	IFF_UNICAST_FLT in its priv_flags.
 *
 * int (*ndo_set_mac_address)(struct net_device *dev, void *addr);
 *	This function  is called when the Media Access Control address
 *	needs to be changed. If this interface is not defined, the
 *	MAC address can not be changed.
 *
 * int (*ndo_validate_addr)(struct net_device *dev);
 *	Test if Media Access Control address is valid for the device.
 *
 * int (*ndo_do_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *	Old-style ioctl entry point. This is used internally by the
 *	appletalk and ieee802154 subsystems but is no longer called by
 *	the device ioctl handler.
 *
 * int (*ndo_siocbond)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *	Used by the bonding driver for its device specific ioctls:
 *	SIOCBONDENSLAVE, SIOCBONDRELEASE, SIOCBONDSETHWADDR, SIOCBONDCHANGEACTIVE,
 *	SIOCBONDSLAVEINFOQUERY, and SIOCBONDINFOQUERY
 *
 * * int (*ndo_eth_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *	Called for ethernet specific ioctls: SIOCGMIIPHY, SIOCGMIIREG,
 *	SIOCSMIIREG, SIOCSHWTSTAMP and SIOCGHWTSTAMP.
 *
 * int (*ndo_set_config)(struct net_device *dev, struct ifmap *map);
 *	Used to set network devices bus interface parameters. This interface
 *	is retained for legacy reasons; new devices should use the bus
 *	interface (PCI) for low level management.
 *
 * int (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
 *	Called when a user wants to change the Maximum Transfer Unit
 *	of a device.
 *
 * void (*ndo_tx_timeout)(struct net_device *dev, unsigned int txqueue);
 *	Callback used when the transmitter has not made any progress
 *	for dev->watchdog ticks.
 *
 * void (*ndo_get_stats64)(struct net_device *dev,
 *                         struct rtnl_link_stats64 *storage);
 * struct net_device_stats* (*ndo_get_stats)(struct net_device *dev);
 *	Called when a user wants to get the network device usage
 *	statistics. Drivers must do one of the following:
 *	1. Define @ndo_get_stats64 to fill in a zero-initialised
 *	   rtnl_link_stats64 structure passed by the caller.
 *	2. Define @ndo_get_stats to update a net_device_stats structure
 *	   (which should normally be dev->stats) and return a pointer to
 *	   it. The structure may be changed asynchronously only if each
 *	   field is written atomically.
 *	3. Update dev->stats asynchronously and atomically, and define
 *	   neither operation.
 *
 * bool (*ndo_has_offload_stats)(const struct net_device *dev, int attr_id)
 *	Return true if this device supports offload stats of this attr_id.
 *
 * int (*ndo_get_offload_stats)(int attr_id, const struct net_device *dev,
 *	void *attr_data)
 *	Get statistics for offload operations by attr_id. Write it into the
 *	attr_data pointer.
 *
 * int (*ndo_vlan_rx_add_vid)(struct net_device *dev, __be16 proto, u16 vid);
 *	If device supports VLAN filtering this function is called when a
 *	VLAN id is registered.
 *
 * int (*ndo_vlan_rx_kill_vid)(struct net_device *dev, __be16 proto, u16 vid);
 *	If device supports VLAN filtering this function is called when a
 *	VLAN id is unregistered.
 *
 * void (*ndo_poll_controller)(struct net_device *dev);
 *
 *	SR-IOV management functions.
 * int (*ndo_set_vf_mac)(struct net_device *dev, int vf, u8* mac);
 * int (*ndo_set_vf_vlan)(struct net_device *dev, int vf, u16 vlan,
 *			  u8 qos, __be16 proto);
 * int (*ndo_set_vf_rate)(struct net_device *dev, int vf, int min_tx_rate,
 *			  int max_tx_rate);
 * int (*ndo_set_vf_spoofchk)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_set_vf_trust)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_get_vf_config)(struct net_device *dev,
 *			    int vf, struct ifla_vf_info *ivf);
 * int (*ndo_set_vf_link_state)(struct net_device *dev, int vf, int link_state);
 * int (*ndo_set_vf_port)(struct net_device *dev, int vf,
 *			  struct nlattr *port[]);
 *
 *      Enable or disable the VF ability to query its RSS Redirection Table and
 *      Hash Key. This is needed since on some devices VF share this information
 *      with PF and querying it may introduce a theoretical security risk.
 * int (*ndo_set_vf_rss_query_en)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_get_vf_port)(struct net_device *dev, int vf, struct sk_buff *skb);
 * int (*ndo_setup_tc)(struct net_device *dev, enum tc_setup_type type,
 *		       void *type_data);
 *	Called to setup any 'tc' scheduler, classifier or action on @dev.
 *	This is always called from the stack with the rtnl lock held and netif
 *	tx queues stopped. This allows the netdevice to perform queue
 *	management safely.
 *
 *	Fiber Channel over Ethernet (FCoE) offload functions.
 * int (*ndo_fcoe_enable)(struct net_device *dev);
 *	Called when the FCoE protocol stack wants to start using LLD for FCoE
 *	so the underlying device can perform whatever needed configuration or
 *	initialization to support acceleration of FCoE traffic.
 *
 * int (*ndo_fcoe_disable)(struct net_device *dev);
 *	Called when the FCoE protocol stack wants to stop using LLD for FCoE
 *	so the underlying device can perform whatever needed clean-ups to
 *	stop supporting acceleration of FCoE traffic.
 *
 * int (*ndo_fcoe_ddp_setup)(struct net_device *dev, u16 xid,
 *			     struct scatterlist *sgl, unsigned int sgc);
 *	Called when the FCoE Initiator wants to initialize an I/O that
 *	is a possible candidate for Direct Data Placement (DDP). The LLD can
 *	perform necessary setup and returns 1 to indicate the device is set up
 *	successfully to perform DDP on this I/O, otherwise this returns 0.
 *
 * int (*ndo_fcoe_ddp_done)(struct net_device *dev,  u16 xid);
 *	Called when the FCoE Initiator/Target is done with the DDPed I/O as
 *	indicated by the FC exchange id 'xid', so the underlying device can
 *	clean up and reuse resources for later DDP requests.
 *
 * int (*ndo_fcoe_ddp_target)(struct net_device *dev, u16 xid,
 *			      struct scatterlist *sgl, unsigned int sgc);
 *	Called when the FCoE Target wants to initialize an I/O that
 *	is a possible candidate for Direct Data Placement (DDP). The LLD can
 *	perform necessary setup and returns 1 to indicate the device is set up
 *	successfully to perform DDP on this I/O, otherwise this returns 0.
 *
 * int (*ndo_fcoe_get_hbainfo)(struct net_device *dev,
 *			       struct netdev_fcoe_hbainfo *hbainfo);
 *	Called when the FCoE Protocol stack wants information on the underlying
 *	device. This information is utilized by the FCoE protocol stack to
 *	register attributes with Fiber Channel management service as per the
 *	FC-GS Fabric Device Management Information(FDMI) specification.
 *
 * int (*ndo_fcoe_get_wwn)(struct net_device *dev, u64 *wwn, int type);
 *	Called when the underlying device wants to override default World Wide
 *	Name (WWN) generation mechanism in FCoE protocol stack to pass its own
 *	World Wide Port Name (WWPN) or World Wide Node Name (WWNN) to the FCoE
 *	protocol stack to use.
 *
 *	RFS acceleration.
 * int (*ndo_rx_flow_steer)(struct net_device *dev, const struct sk_buff *skb,
 *			    u16 rxq_index, u32 flow_id);
 *	Set hardware filter for RFS.  rxq_index is the target queue index;
 *	flow_id is a flow ID to be passed to rps_may_expire_flow() later.
 *	Return the filter ID on success, or a negative error code.
 *
 *	Slave management functions (for bridge, bonding, etc).
 * int (*ndo_add_slave)(struct net_device *dev, struct net_device *slave_dev);
 *	Called to make another netdev an underling.
 *
 * int (*ndo_del_slave)(struct net_device *dev, struct net_device *slave_dev);
 *	Called to release previously enslaved netdev.
 *
 * struct net_device *(*ndo_get_xmit_slave)(struct net_device *dev,
 *					    struct sk_buff *skb,
 *					    bool all_slaves);
 *	Get the xmit slave of master device. If all_slaves is true, function
 *	assume all the slaves can transmit.
 *
 *      Feature/offload setting functions.
 * netdev_features_t (*ndo_fix_features)(struct net_device *dev,
 *		netdev_features_t features);
 *	Adjusts the requested feature flags according to device-specific
 *	constraints, and returns the resulting flags. Must not modify
 *	the device state.
 *
 * int (*ndo_set_features)(struct net_device *dev, netdev_features_t features);
 *	Called to update device configuration to new features. Passed
 *	feature set might be less than what was returned by ndo_fix_features()).
 *	Must return >0 or -errno if it changed dev->features itself.
 *
 * int (*ndo_fdb_add)(struct ndmsg *ndm, struct nlattr *tb[],
 *		      struct net_device *dev,
 *		      const unsigned char *addr, u16 vid, u16 flags,
 *		      struct netlink_ext_ack *extack);
 *	Adds an FDB entry to dev for addr.
 * int (*ndo_fdb_del)(struct ndmsg *ndm, struct nlattr *tb[],
 *		      struct net_device *dev,
 *		      const unsigned char *addr, u16 vid)
 *	Deletes the FDB entry from dev coresponding to addr.
 * int (*ndo_fdb_dump)(struct sk_buff *skb, struct netlink_callback *cb,
 *		       struct net_device *dev, struct net_device *filter_dev,
 *		       int *idx)
 *	Used to add FDB entries to dump requests. Implementers should add
 *	entries to skb and update idx with the number of entries.
 *
 * int (*ndo_bridge_setlink)(struct net_device *dev, struct nlmsghdr *nlh,
 *			     u16 flags, struct netlink_ext_ack *extack)
 * int (*ndo_bridge_getlink)(struct sk_buff *skb, u32 pid, u32 seq,
 *			     struct net_device *dev, u32 filter_mask,
 *			     int nlflags)
 * int (*ndo_bridge_dellink)(struct net_device *dev, struct nlmsghdr *nlh,
 *			     u16 flags);
 *
 * int (*ndo_change_carrier)(struct net_device *dev, bool new_carrier);
 *	Called to change device carrier. Soft-devices (like dummy, team, etc)
 *	which do not represent real hardware may define this to allow their
 *	userspace components to manage their virtual carrier state. Devices
 *	that determine carrier state from physical hardware properties (eg
 *	network cables) or protocol-dependent mechanisms (eg
 *	USB_CDC_NOTIFY_NETWORK_CONNECTION) should NOT implement this function.
 *
 * int (*ndo_get_phys_port_id)(struct net_device *dev,
 *			       struct netdev_phys_item_id *ppid);
 *	Called to get ID of physical port of this device. If driver does
 *	not implement this, it is assumed that the hw is not able to have
 *	multiple net devices on single physical port.
 *
 * int (*ndo_get_port_parent_id)(struct net_device *dev,
 *				 struct netdev_phys_item_id *ppid)
 *	Called to get the parent ID of the physical port of this device.
 *
 * void* (*ndo_dfwd_add_station)(struct net_device *pdev,
 *				 struct net_device *dev)
 *	Called by upper layer devices to accelerate switching or other
 *	station functionality into hardware. 'pdev is the lowerdev
 *	to use for the offload and 'dev' is the net device that will
 *	back the offload. Returns a pointer to the private structure
 *	the upper layer will maintain.
 * void (*ndo_dfwd_del_station)(struct net_device *pdev, void *priv)
 *	Called by upper layer device to delete the station created
 *	by 'ndo_dfwd_add_station'. 'pdev' is the net device backing
 *	the station and priv is the structure returned by the add
 *	operation.
 * int (*ndo_set_tx_maxrate)(struct net_device *dev,
 *			     int queue_index, u32 maxrate);
 *	Called when a user wants to set a max-rate limitation of specific
 *	TX queue.
 * int (*ndo_get_iflink)(const struct net_device *dev);
 *	Called to get the iflink value of this device.
 * void (*ndo_change_proto_down)(struct net_device *dev,
 *				 bool proto_down);
 *	This function is used to pass protocol port error state information
 *	to the switch driver. The switch driver can react to the proto_down
 *      by doing a phys down on the associated switch port.
 * int (*ndo_fill_metadata_dst)(struct net_device *dev, struct sk_buff *skb);
 *	This function is used to get egress tunnel information for given skb.
 *	This is useful for retrieving outer tunnel header parameters while
 *	sampling packet.
 * void (*ndo_set_rx_headroom)(struct net_device *dev, int needed_headroom);
 *	This function is used to specify the headroom that the skb must
 *	consider when allocation skb during packet reception. Setting
 *	appropriate rx headroom value allows avoiding skb head copy on
 *	forward. Setting a negative value resets the rx headroom to the
 *	default value.
 * int (*ndo_bpf)(struct net_device *dev, struct netdev_bpf *bpf);
 *	This function is used to set or query state related to XDP on the
 *	netdevice and manage BPF offload. See definition of
 *	enum bpf_netdev_command for details.
 * int (*ndo_xdp_xmit)(struct net_device *dev, int n, struct xdp_frame **xdp,
 *			u32 flags);
 *	This function is used to submit @n XDP packets for transmit on a
 *	netdevice. Returns number of frames successfully transmitted, frames
 *	that got dropped are freed/returned via xdp_return_frame().
 *	Returns negative number, means general error invoking ndo, meaning
 *	no frames were xmit'ed and core-caller will free all frames.
 * struct net_device *(*ndo_xdp_get_xmit_slave)(struct net_device *dev,
 *					        struct xdp_buff *xdp);
 *      Get the xmit slave of master device based on the xdp_buff.
 * int (*ndo_xsk_wakeup)(struct net_device *dev, u32 queue_id, u32 flags);
 *      This function is used to wake up the softirq, ksoftirqd or kthread
 *	responsible for sending and/or receiving packets on a specific
 *	queue id bound to an AF_XDP socket. The flags field specifies if
 *	only RX, only Tx, or both should be woken up using the flags
 *	XDP_WAKEUP_RX and XDP_WAKEUP_TX.
 * struct devlink_port *(*ndo_get_devlink_port)(struct net_device *dev);
 *	Get devlink port instance associated with a given netdev.
 *	Called with a reference on the netdevice and devlink locks only,
 *	rtnl_lock is not held.
 * int (*ndo_tunnel_ctl)(struct net_device *dev, struct ip_tunnel_parm *p,
 *			 int cmd);
 *	Add, change, delete or get information on an IPv4 tunnel.
 * struct net_device *(*ndo_get_peer_dev)(struct net_device *dev);
 *	If a device is paired with a peer device, return the peer instance.
 *	The caller must be under RCU read context.
 * int (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx, struct net_device_path *path);
 *     Get the forwarding path to reach the real device from the HW destination address
 */
struct net_device_ops {
	int			(*ndo_init)(struct net_device *dev);
	void			(*ndo_uninit)(struct net_device *dev);
	int			(*ndo_open)(struct net_device *dev);
	int			(*ndo_stop)(struct net_device *dev);
	netdev_tx_t		(*ndo_start_xmit)(struct sk_buff *skb,
						  struct net_device *dev);
	netdev_features_t	(*ndo_features_check)(struct sk_buff *skb,
						      struct net_device *dev,
						      netdev_features_t features);
	u16			(*ndo_select_queue)(struct net_device *dev,
						    struct sk_buff *skb,
						    struct net_device *sb_dev);
	void			(*ndo_change_rx_flags)(struct net_device *dev,
						       int flags);
	void			(*ndo_set_rx_mode)(struct net_device *dev);
	int			(*ndo_set_mac_address)(struct net_device *dev,
						       void *addr);
	int			(*ndo_validate_addr)(struct net_device *dev);
	int			(*ndo_do_ioctl)(struct net_device *dev,
					        struct ifreq *ifr, int cmd);
	int			(*ndo_eth_ioctl)(struct net_device *dev,
						 struct ifreq *ifr, int cmd);
	int			(*ndo_siocbond)(struct net_device *dev,
						struct ifreq *ifr, int cmd);
	int			(*ndo_siocwandev)(struct net_device *dev,
						  struct if_settings *ifs);
	int			(*ndo_siocdevprivate)(struct net_device *dev,
						      struct ifreq *ifr,
						      void __user *data, int cmd);
	int			(*ndo_set_config)(struct net_device *dev,
					          struct ifmap *map);
	int			(*ndo_change_mtu)(struct net_device *dev,
						  int new_mtu);
	int			(*ndo_neigh_setup)(struct net_device *dev,
						   struct neigh_parms *);
	void			(*ndo_tx_timeout) (struct net_device *dev,
						   unsigned int txqueue);

	void			(*ndo_get_stats64)(struct net_device *dev,
						   struct rtnl_link_stats64 *storage);
	bool			(*ndo_has_offload_stats)(const struct net_device *dev, int attr_id);
	int			(*ndo_get_offload_stats)(int attr_id,
							 const struct net_device *dev,
							 void *attr_data);
	struct net_device_stats* (*ndo_get_stats)(struct net_device *dev);

	int			(*ndo_vlan_rx_add_vid)(struct net_device *dev,
						       __be16 proto, u16 vid);
	int			(*ndo_vlan_rx_kill_vid)(struct net_device *dev,
						        __be16 proto, u16 vid);
#ifdef CONFIG_NET_POLL_CONTROLLER
	void                    (*ndo_poll_controller)(struct net_device *dev);
	int			(*ndo_netpoll_setup)(struct net_device *dev,
						     struct netpoll_info *info);
	void			(*ndo_netpoll_cleanup)(struct net_device *dev);
#endif
	int			(*ndo_set_vf_mac)(struct net_device *dev,
						  int queue, u8 *mac);
	int			(*ndo_set_vf_vlan)(struct net_device *dev,
						   int queue, u16 vlan,
						   u8 qos, __be16 proto);
	int			(*ndo_set_vf_rate)(struct net_device *dev,
						   int vf, int min_tx_rate,
						   int max_tx_rate);
	int			(*ndo_set_vf_spoofchk)(struct net_device *dev,
						       int vf, bool setting);
	int			(*ndo_set_vf_trust)(struct net_device *dev,
						    int vf, bool setting);
	int			(*ndo_get_vf_config)(struct net_device *dev,
						     int vf,
						     struct ifla_vf_info *ivf);
	int			(*ndo_set_vf_link_state)(struct net_device *dev,
							 int vf, int link_state);
	int			(*ndo_get_vf_stats)(struct net_device *dev,
						    int vf,
						    struct ifla_vf_stats
						    *vf_stats);
	int			(*ndo_set_vf_port)(struct net_device *dev,
						   int vf,
						   struct nlattr *port[]);
	int			(*ndo_get_vf_port)(struct net_device *dev,
						   int vf, struct sk_buff *skb);
	int			(*ndo_get_vf_guid)(struct net_device *dev,
						   int vf,
						   struct ifla_vf_guid *node_guid,
						   struct ifla_vf_guid *port_guid);
	int			(*ndo_set_vf_guid)(struct net_device *dev,
						   int vf, u64 guid,
						   int guid_type);
	int			(*ndo_set_vf_rss_query_en)(
						   struct net_device *dev,
						   int vf, bool setting);
	int			(*ndo_setup_tc)(struct net_device *dev,
						enum tc_setup_type type,
						void *type_data);
#if IS_ENABLED(CONFIG_FCOE)
	int			(*ndo_fcoe_enable)(struct net_device *dev);
	int			(*ndo_fcoe_disable)(struct net_device *dev);
	int			(*ndo_fcoe_ddp_setup)(struct net_device *dev,
						      u16 xid,
						      struct scatterlist *sgl,
						      unsigned int sgc);
	int			(*ndo_fcoe_ddp_done)(struct net_device *dev,
						     u16 xid);
	int			(*ndo_fcoe_ddp_target)(struct net_device *dev,
						       u16 xid,
						       struct scatterlist *sgl,
						       unsigned int sgc);
	int			(*ndo_fcoe_get_hbainfo)(struct net_device *dev,
							struct netdev_fcoe_hbainfo *hbainfo);
#endif

#if IS_ENABLED(CONFIG_LIBFCOE)
#define NETDEV_FCOE_WWNN 0
#define NETDEV_FCOE_WWPN 1
	int			(*ndo_fcoe_get_wwn)(struct net_device *dev,
						    u64 *wwn, int type);
#endif

#ifdef CONFIG_RFS_ACCEL
	int			(*ndo_rx_flow_steer)(struct net_device *dev,
						     const struct sk_buff *skb,
						     u16 rxq_index,
						     u32 flow_id);
#endif
	int			(*ndo_add_slave)(struct net_device *dev,
						 struct net_device *slave_dev,
						 struct netlink_ext_ack *extack);
	int			(*ndo_del_slave)(struct net_device *dev,
						 struct net_device *slave_dev);
	struct net_device*	(*ndo_get_xmit_slave)(struct net_device *dev,
						      struct sk_buff *skb,
						      bool all_slaves);
	struct net_device*	(*ndo_sk_get_lower_dev)(struct net_device *dev,
							struct sock *sk);
	netdev_features_t	(*ndo_fix_features)(struct net_device *dev,
						    netdev_features_t features);
	int			(*ndo_set_features)(struct net_device *dev,
						    netdev_features_t features);
	int			(*ndo_neigh_construct)(struct net_device *dev,
						       struct neighbour *n);
	void			(*ndo_neigh_destroy)(struct net_device *dev,
						     struct neighbour *n);

	int			(*ndo_fdb_add)(struct ndmsg *ndm,
					       struct nlattr *tb[],
					       struct net_device *dev,
					       const unsigned char *addr,
					       u16 vid,
					       u16 flags,
					       struct netlink_ext_ack *extack);
	int			(*ndo_fdb_del)(struct ndmsg *ndm,
					       struct nlattr *tb[],
					       struct net_device *dev,
					       const unsigned char *addr,
					       u16 vid);
	int			(*ndo_fdb_dump)(struct sk_buff *skb,
						struct netlink_callback *cb,
						struct net_device *dev,
						struct net_device *filter_dev,
						int *idx);
	int			(*ndo_fdb_get)(struct sk_buff *skb,
					       struct nlattr *tb[],
					       struct net_device *dev,
					       const unsigned char *addr,
					       u16 vid, u32 portid, u32 seq,
					       struct netlink_ext_ack *extack);
	int			(*ndo_bridge_setlink)(struct net_device *dev,
						      struct nlmsghdr *nlh,
						      u16 flags,
						      struct netlink_ext_ack *extack);
	int			(*ndo_bridge_getlink)(struct sk_buff *skb,
						      u32 pid, u32 seq,
						      struct net_device *dev,
						      u32 filter_mask,
						      int nlflags);
	int			(*ndo_bridge_dellink)(struct net_device *dev,
						      struct nlmsghdr *nlh,
						      u16 flags);
	int			(*ndo_change_carrier)(struct net_device *dev,
						      bool new_carrier);
	int			(*ndo_get_phys_port_id)(struct net_device *dev,
							struct netdev_phys_item_id *ppid);
	int			(*ndo_get_port_parent_id)(struct net_device *dev,
							  struct netdev_phys_item_id *ppid);
	int			(*ndo_get_phys_port_name)(struct net_device *dev,
							  char *name, size_t len);
	void*			(*ndo_dfwd_add_station)(struct net_device *pdev,
							struct net_device *dev);
	void			(*ndo_dfwd_del_station)(struct net_device *pdev,
							void *priv);

	int			(*ndo_set_tx_maxrate)(struct net_device *dev,
						      int queue_index,
						      u32 maxrate);
	int			(*ndo_get_iflink)(const struct net_device *dev);
	int			(*ndo_change_proto_down)(struct net_device *dev,
							 bool proto_down);
	int			(*ndo_fill_metadata_dst)(struct net_device *dev,
						       struct sk_buff *skb);
	void			(*ndo_set_rx_headroom)(struct net_device *dev,
						       int needed_headroom);
	int			(*ndo_bpf)(struct net_device *dev,
					   struct netdev_bpf *bpf);
	int			(*ndo_xdp_xmit)(struct net_device *dev, int n,
						struct xdp_frame **xdp,
						u32 flags);
	struct net_device *	(*ndo_xdp_get_xmit_slave)(struct net_device *dev,
							  struct xdp_buff *xdp);
	int			(*ndo_xsk_wakeup)(struct net_device *dev,
						  u32 queue_id, u32 flags);
	struct devlink_port *	(*ndo_get_devlink_port)(struct net_device *dev);
	int			(*ndo_tunnel_ctl)(struct net_device *dev,
						  struct ip_tunnel_parm *p, int cmd);
	struct net_device *	(*ndo_get_peer_dev)(struct net_device *dev);
	int                     (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx,
                                                         struct net_device_path *path);
};
````

#### ndo_open

打开网络设备的时候此函数会执行，网络驱动程序需要实现此函数，非常重要！一般会在此函数中做如下工作：

* 使能网络外设时钟；
* 申请网络所使用的环形缓冲区；
* 初始化MAC外设；
* 绑定对于的PHY；
* 如果使用NAPI需要使能NAPI模块，通过 `napi_enable` 函数使能；
* 开启PHY；
* 调用 `netif_tx_start_all_queues` 来使能传输队列，也可能调用 `netif_start_queue` 函数。

#### ndo_stop

用于关闭网络设备，一般会进行以下操作：

* 停止PHY;
* 停止NAPI功能；
* 停止发送功能；
* 关闭MAC；
* 断开PHY连接；
* 关闭网络时钟；
* 释放数据缓冲区；

#### ndo_start_xmit

用于执行数据包发送功能，参数中 `sk_buff` 结构体指针在Linux网络驱动中相当重要，其中保存了上层传递给网络驱动层的数据；

#### ndo_select_queue

当设备支持多个队列时现在使用哪个队列传输。

#### ndo_tx_timeout

超时函数；当发送超时的时候产生会执行，一般都是网络出问题了导致发送超时。一般可能会重启 MAC 和 PHY，重新开始数据发送等。

#### ndo_set_rx_mode

配置网卡接收模式，还用于改变地址过滤列表；

#### **ndo_set_features** 

修改 net_device 的 features 属性，设置相应的硬件属性。

#### ndo_validate_addr

验证 MAC 地址是否合法，也即是验证 net_device 的dev_addr 中的 MAC 地址是否合法，直接调用 is_valid_ether_addr 函数。

#### ndo_set_mac_address

此函数用于修改网卡的 MAC 地址，设置 net_device的 dev_addr 成员变量， 并且将 MAC 地址写入到网络外设的硬件寄存器中。

#### ndo_poll_controller

使用查询方式来处理网卡数据的收发。

#### **ndo_do_ioctl** 

用户程序调用 `ioctl` 的时候此函数就会执行，比如 PHY 芯片相关的命令操作，一般会直接调用 phy_mii_ioctl 函数。

#### **ndo_change_mtu** 

更改MTU大小；

