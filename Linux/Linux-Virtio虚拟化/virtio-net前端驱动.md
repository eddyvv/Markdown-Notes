# virtio-net 前端驱动

`virtio` 前端驱动主要完成对 `net_device_ops` 结构体和 `virtio_driver` 结构体成员的赋值。`net_device_ops` 是一个网络设备的操作集合，它定义了网络设备的常见操作函数指针，例如发送数据、接收数据、打开和关闭设备等操作。`virtio` 前端驱动通过一个 `virtio_driver` 结构体来维护虚拟网卡设备的基本功能、状态信息和进行 `I/O` 操作。

## struct net_device_ops

`源码位置`[/include/linux/netdevice.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/netdevice.h)

```c
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
 * int (*ndo_fdb_del_bulk)(struct ndmsg *ndm, struct nlattr *tb[],
 *			   struct net_device *dev,
 *			   u16 vid,
 *			   struct netlink_ext_ack *extack);
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
 * int (*ndo_tunnel_ctl)(struct net_device *dev, struct ip_tunnel_parm *p,
 *			 int cmd);
 *	Add, change, delete or get information on an IPv4 tunnel.
 * struct net_device *(*ndo_get_peer_dev)(struct net_device *dev);
 *	If a device is paired with a peer device, return the peer instance.
 *	The caller must be under RCU read context.
 * int (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx, struct net_device_path *path);
 *     Get the forwarding path to reach the real device from the HW destination address
 * ktime_t (*ndo_get_tstamp)(struct net_device *dev,
 *			     const struct skb_shared_hwtstamps *hwtstamps,
 *			     bool cycles);
 *	Get hardware timestamp based on normal/adjustable time or free running
 *	cycle counter. This function is required if physical clock supports a
 *	free running cycle counter.
 */
struct net_device_ops {
    			/* 当第一次注册网络设备的时候此函数会执行 */
	int			(*ndo_init)(struct net_device *dev);
    				/* 卸载网络设备的时候此函数会执行 */
	void			(*ndo_uninit)(struct net_device *dev);
    			/* 打开网络设备时调用此函数，非常重要 */
	int			(*ndo_open)(struct net_device *dev);
    			/* 关闭网络设备时调用 */
	int			(*ndo_stop)(struct net_device *dev);
    				/* 发送数据 */
	netdev_tx_t		(*ndo_start_xmit)(struct sk_buff *skb,
						  struct net_device *dev);
	netdev_features_t	(*ndo_features_check)(struct sk_buff *skb,
						      struct net_device *dev,
						      netdev_features_t features);
    			/* 当设备支持多传输队列的时候选择使用哪个队列。 */
	u16			(*ndo_select_queue)(struct net_device *dev,
						    struct sk_buff *skb,
						    struct net_device *sb_dev);
    				/* 设备在启用多播或混杂时更改配置。 */
	void			(*ndo_change_rx_flags)(struct net_device *dev,
						       int flags);
    				/* 配置网卡接收模式，还用于设置地址过滤列表 */
	void			(*ndo_set_rx_mode)(struct net_device *dev);
    			/* 修改网卡 MAC 地址 */
	int			(*ndo_set_mac_address)(struct net_device *dev,
						       void *addr);
    			/* 验证 MAC 地址是否合法，也即是验证 net_device 的dev_addr 中的 MAC 地址是否合法，直接调用 is_valid_ether_addr 函数 */
	int			(*ndo_validate_addr)(struct net_device *dev);
    			/* 用户程序调用 ioctl 的时候此函数就会执行，比如 PHY 芯片相关的命令操作，一般会直接调用 phy_mii_ioctl 函数。 */
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
    			/* 设置网络设备总线接口参数，目前已不再使用 */
	int			(*ndo_set_config)(struct net_device *dev,
					          struct ifmap *map);
    			/* 更改硬件的MTU值以及是否需要重新分配初始化描述符环以及接收缓存 */
	int			(*ndo_change_mtu)(struct net_device *dev,
						  int new_mtu);
	int			(*ndo_neigh_setup)(struct net_device *dev,
						   struct neigh_parms *);
    				/* 超时函数；当发送超时的时候会执行，一般都是网络出问题了导致发送超时，可能会重启 MAC 和 PHY，重新开始数据发送 */
	void			(*ndo_tx_timeout) (struct net_device *dev,
						   unsigned int txqueue);
					/* 拷贝硬件统计计数到用户空间，注意使用的是__u64；对于1G或者更快的网卡，64位是必须的； */
	void			(*ndo_get_stats64)(struct net_device *dev,
						   struct rtnl_link_stats64 *storage);
    				/* 判断硬件是否支持 attr_id 的硬件卸载功能 */
	bool			(*ndo_has_offload_stats)(const struct net_device *dev, int attr_id);
    			/* 通过 attr_id 获取卸载操作的统计信息。将其写入 attr_data 指针。 */
	int			(*ndo_get_offload_stats)(int attr_id,
							 const struct net_device *dev,
							 void *attr_data);
    						/* 函数用于获得网络设备的状态信息 */
	struct net_device_stats* (*ndo_get_stats)(struct net_device *dev);
				/* 当设备支持 VLAN 过滤的时候，这个函数在一个 VLAN ID 被注册时候调用  */
	int			(*ndo_vlan_rx_add_vid)(struct net_device *dev,
						       __be16 proto, u16 vid);
    			/* 当设备支持 VLAN 过滤的时候，这个函数在一个 VLAN ID 被注销时候调用 */
	int			(*ndo_vlan_rx_kill_vid)(struct net_device *dev,
						        __be16 proto, u16 vid);
#ifdef CONFIG_NET_POLL_CONTROLLER
    						/* 使用查询方式来处理网卡数据的收发 */
	void                    (*ndo_poll_controller)(struct net_device *dev);
	int			(*ndo_netpoll_setup)(struct net_device *dev,
						     struct netpoll_info *info);
	void			(*ndo_netpoll_cleanup)(struct net_device *dev);
#endif
    	/* SR-IOV相关管理函数 */
    			/* 设置虚拟网卡的 MAC 地址 */
	int			(*ndo_set_vf_mac)(struct net_device *dev,
						  int queue, u8 *mac);
    			/* 设置虚拟网卡的 VLAN ID 和 User Priority */
	int			(*ndo_set_vf_vlan)(struct net_device *dev,
						   int queue, u16 vlan,
						   u8 qos, __be16 proto);
    			/* 设置虚拟网卡的传输速率，由于多个虚拟网卡共享一个物理端口，所以单个虚拟网卡的传输速率肯定小于标准值 */
	int			(*ndo_set_vf_rate)(struct net_device *dev,
						   int vf, int min_tx_rate,
						   int max_tx_rate);
    			/* 设置虚拟网卡的 MAC/VLAN 反欺诈过滤功能 */
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
    			/* 修改 net_device 的 features 属性，设置相应的硬件属性 */
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
					       u16 vid, struct netlink_ext_ack *extack);
	int			(*ndo_fdb_del_bulk)(struct ndmsg *ndm,
						    struct nlattr *tb[],
						    struct net_device *dev,
						    u16 vid,
						    struct netlink_ext_ack *extack);
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
	int			(*ndo_tunnel_ctl)(struct net_device *dev,
						  struct ip_tunnel_parm *p, int cmd);
	struct net_device *	(*ndo_get_peer_dev)(struct net_device *dev);
	int                     (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx,
                                                         struct net_device_path *path);
	ktime_t			(*ndo_get_tstamp)(struct net_device *dev,
						  const struct skb_shared_hwtstamps *hwtstamps,
						  bool cycles);
};
```

### virtnet_netdev

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static const struct net_device_ops virtnet_netdev = {
	.ndo_open            = virtnet_open,
	.ndo_stop   	     = virtnet_close,
	.ndo_start_xmit      = start_xmit,
	.ndo_validate_addr   = eth_validate_addr,
	.ndo_set_mac_address = virtnet_set_mac_address,
	.ndo_set_rx_mode     = virtnet_set_rx_mode,
	.ndo_get_stats64     = virtnet_stats,
	.ndo_vlan_rx_add_vid = virtnet_vlan_rx_add_vid,
	.ndo_vlan_rx_kill_vid = virtnet_vlan_rx_kill_vid,
	.ndo_bpf		= virtnet_xdp,
	.ndo_xdp_xmit		= virtnet_xdp_xmit,
	.ndo_features_check	= passthru_features_check,
	.ndo_get_phys_port_name	= virtnet_get_phys_port_name,
	.ndo_set_features	= virtnet_set_features,
	.ndo_tx_timeout		= virtnet_tx_timeout,
};
```

#### virtnet_open

打开网络设备时执行，一般会在此函数中做如下工作：

* 使能网络外设时钟；
* 申请网络所使用的环形缓冲区；
* 初始化MAC外设；
* 绑定对于的PHY；
* 如果使用NAPI需要使能NAPI模块，通过 `napi_enable` 函数使能；
* 开启PHY；
* 调用 `netif_tx_start_all_queues` 来使能传输队列，也可能调用 `netif_start_queue` 函数。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_open(struct net_device *dev)
{
	struct virtnet_info *vi = netdev_priv(dev);
	int i, err;

	enable_delayed_refill(vi);

	for (i = 0; i < vi->max_queue_pairs; i++) {
		if (i < vi->curr_queue_pairs)
			/* Make sure we have some buffers: if oom use wq. */
			if (!try_fill_recv(vi, &vi->rq[i], GFP_KERNEL))
				schedule_delayed_work(&vi->refill, 0);

		err = xdp_rxq_info_reg(&vi->rq[i].xdp_rxq, dev, i, vi->rq[i].napi.napi_id);
		if (err < 0)
			return err;

		err = xdp_rxq_info_reg_mem_model(&vi->rq[i].xdp_rxq,
						 MEM_TYPE_PAGE_SHARED, NULL);
		if (err < 0) {
			xdp_rxq_info_unreg(&vi->rq[i].xdp_rxq);
			return err;
		}

		virtnet_napi_enable(vi->rq[i].vq, &vi->rq[i].napi);
		virtnet_napi_tx_enable(vi, vi->sq[i].vq, &vi->sq[i].napi);
	}

	return 0;
}
```

#### virtnet_close

当网络设备转换到关闭状态时调用此函数。

用于关闭网络设备，一般会进行以下操作：

* 停止PHY;
* 停止NAPI功能；
* 停止发送功能；
* 关闭MAC；
* 断开PHY连接；
* 关闭网络时钟；
* 释放数据缓冲区；

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_close(struct net_device *dev)
{
	struct virtnet_info *vi = netdev_priv(dev);
	int i;

	/* Make sure NAPI doesn't schedule refill work */
	disable_delayed_refill(vi);
	/* Make sure refill_work doesn't re-enable napi! */
	cancel_delayed_work_sync(&vi->refill);

	for (i = 0; i < vi->max_queue_pairs; i++) {
		xdp_rxq_info_unreg(&vi->rq[i].xdp_rxq);
		napi_disable(&vi->rq[i].napi);
		virtnet_napi_tx_disable(&vi->sq[i].napi);
	}

	return 0;
}
```

#### start_xmit

`start_xmit` 在需要发送数据的时候执行，此函数有一个参数为 `sk_buff` 结构体指针，`sk_buff` 结构体在Linux的网络驱动中非常重要，`sk_buff` 保存了上层传递给网络驱动层的数据。即要发送出去的数据都存在了 `sk_buff` 中，`skb` 的数据都是零拷贝的方式交给后端驱动。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
{
	struct virtnet_info *vi = netdev_priv(dev);
	int qnum = skb_get_queue_mapping(skb);
	struct send_queue *sq = &vi->sq[qnum];
	int err;
	struct netdev_queue *txq = netdev_get_tx_queue(dev, qnum);
	bool kick = !netdev_xmit_more();
	bool use_napi = sq->napi.weight;

	/* Free up any pending old buffers before queueing new ones. */
	do {
		if (use_napi)
			virtqueue_disable_cb(sq->vq);

		free_old_xmit_skbs(sq, false);

	} while (use_napi && kick &&
	       unlikely(!virtqueue_enable_cb_delayed(sq->vq)));

	/* timestamp packet in software */
	skb_tx_timestamp(skb);

	/* Try to transmit */
	err = xmit_skb(sq, skb);

	/* This should not happen! */
	if (unlikely(err)) {
		dev->stats.tx_fifo_errors++;
		if (net_ratelimit())
			dev_warn(&dev->dev,
				 "Unexpected TXQ (%d) queue failure: %d\n",
				 qnum, err);
		dev->stats.tx_dropped++;
		dev_kfree_skb_any(skb);
		return NETDEV_TX_OK;
	}

	/* Don't wait up for transmitted skbs to be freed. */
	if (!use_napi) {
		skb_orphan(skb);
		nf_reset_ct(skb);
	}

	/* If running out of space, stop queue to avoid getting packets that we
	 * are then unable to transmit.
	 * An alternative would be to force queuing layer to requeue the skb by
	 * returning NETDEV_TX_BUSY. However, NETDEV_TX_BUSY should not be
	 * returned in a normal path of operation: it means that driver is not
	 * maintaining the TX queue stop/start state properly, and causes
	 * the stack to do a non-trivial amount of useless work.
	 * Since most packets only take 1 or 2 ring slots, stopping the queue
	 * early means 16 slots are typically wasted.
	 */
	if (sq->vq->num_free < 2+MAX_SKB_FRAGS) {
		netif_stop_subqueue(dev, qnum);
		if (!use_napi &&
		    unlikely(!virtqueue_enable_cb_delayed(sq->vq))) {
			/* More just got used, free them then recheck. */
			free_old_xmit_skbs(sq, false);
			if (sq->vq->num_free >= 2+MAX_SKB_FRAGS) {
				netif_start_subqueue(dev, qnum);
				virtqueue_disable_cb(sq->vq);
			}
		}
	}

	if (kick || netif_xmit_stopped(txq)) {
		if (virtqueue_kick_prepare(sq->vq) && virtqueue_notify(sq->vq)) {
			u64_stats_update_begin(&sq->stats.syncp);
			sq->stats.kicks++;
			u64_stats_update_end(&sq->stats.syncp);
		}
	}

	return NETDEV_TX_OK;
}
```

#### eth_validate_addr

测试网络访问控制地址是否对设备有效。

`源码位置`[/net/ethernet/eth.c](https://github1s.com/torvalds/linux/blob/HEAD/net/ethernet/eth.c)

```c
int eth_validate_addr(struct net_device *dev)
{
	if (!is_valid_ether_addr(dev->dev_addr))
		return -EADDRNOTAVAIL;

	return 0;
}
EXPORT_SYMBOL(eth_validate_addr);
```

#### virtnet_set_mac_address

更改 `MAC` 地址调用

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_set_mac_address(struct net_device *dev, void *p)
{
	struct virtnet_info *vi = netdev_priv(dev);
	struct virtio_device *vdev = vi->vdev;
	int ret;
	struct sockaddr *addr;
	struct scatterlist sg;

	if (virtio_has_feature(vi->vdev, VIRTIO_NET_F_STANDBY))
		return -EOPNOTSUPP;

	addr = kmemdup(p, sizeof(*addr), GFP_KERNEL);
	if (!addr)
		return -ENOMEM;

	ret = eth_prepare_mac_addr_change(dev, addr);
	if (ret)
		goto out;

	if (virtio_has_feature(vdev, VIRTIO_NET_F_CTRL_MAC_ADDR)) {
		sg_init_one(&sg, addr->sa_data, dev->addr_len);
		if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_MAC,
					  VIRTIO_NET_CTRL_MAC_ADDR_SET, &sg)) {
			dev_warn(&vdev->dev,
				 "Failed to set mac address by vq command.\n");
			ret = -EINVAL;
			goto out;
		}
	} else if (virtio_has_feature(vdev, VIRTIO_NET_F_MAC) &&
		   !virtio_has_feature(vdev, VIRTIO_F_VERSION_1)) {
		unsigned int i;

		/* Naturally, this has an atomicity problem. */
		for (i = 0; i < dev->addr_len; i++)
			virtio_cwrite8(vdev,
				       offsetof(struct virtio_net_config, mac) +
				       i, addr->sa_data[i]);
	}

	eth_commit_mac_addr_change(dev, p);
	ret = 0;

out:
	kfree(addr);
	return ret;
}

```

#### virtnet_set_rx_mode

设备更改地址列表过滤

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c\
static void virtnet_set_rx_mode(struct net_device *dev)
{
	struct virtnet_info *vi = netdev_priv(dev);
	struct scatterlist sg[2];
	struct virtio_net_ctrl_mac *mac_data;
	struct netdev_hw_addr *ha;
	int uc_count;
	int mc_count;
	void *buf;
	int i;

	/* We can't dynamically set ndo_set_rx_mode, so return gracefully */
	if (!virtio_has_feature(vi->vdev, VIRTIO_NET_F_CTRL_RX))
		return;

	vi->ctrl->promisc = ((dev->flags & IFF_PROMISC) != 0);
	vi->ctrl->allmulti = ((dev->flags & IFF_ALLMULTI) != 0);

	sg_init_one(sg, &vi->ctrl->promisc, sizeof(vi->ctrl->promisc));

	if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_RX,
				  VIRTIO_NET_CTRL_RX_PROMISC, sg))
		dev_warn(&dev->dev, "Failed to %sable promisc mode.\n",
			 vi->ctrl->promisc ? "en" : "dis");

	sg_init_one(sg, &vi->ctrl->allmulti, sizeof(vi->ctrl->allmulti));

	if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_RX,
				  VIRTIO_NET_CTRL_RX_ALLMULTI, sg))
		dev_warn(&dev->dev, "Failed to %sable allmulti mode.\n",
			 vi->ctrl->allmulti ? "en" : "dis");

	uc_count = netdev_uc_count(dev);
	mc_count = netdev_mc_count(dev);
	/* MAC filter - use one buffer for both lists */
	buf = kzalloc(((uc_count + mc_count) * ETH_ALEN) +
		      (2 * sizeof(mac_data->entries)), GFP_ATOMIC);
	mac_data = buf;
	if (!buf)
		return;

	sg_init_table(sg, 2);

	/* Store the unicast list and count in the front of the buffer */
	mac_data->entries = cpu_to_virtio32(vi->vdev, uc_count);
	i = 0;
	netdev_for_each_uc_addr(ha, dev)
		memcpy(&mac_data->macs[i++][0], ha->addr, ETH_ALEN);

	sg_set_buf(&sg[0], mac_data,
		   sizeof(mac_data->entries) + (uc_count * ETH_ALEN));

	/* multicast list and count fill the end */
	mac_data = (void *)&mac_data->macs[uc_count][0];

	mac_data->entries = cpu_to_virtio32(vi->vdev, mc_count);
	i = 0;
	netdev_for_each_mc_addr(ha, dev)
		memcpy(&mac_data->macs[i++][0], ha->addr, ETH_ALEN);

	sg_set_buf(&sg[1], mac_data,
		   sizeof(mac_data->entries) + (mc_count * ETH_ALEN));

	if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_MAC,
				  VIRTIO_NET_CTRL_MAC_TABLE_SET, sg))
		dev_warn(&dev->dev, "Failed to set MAC filter table.\n");

	kfree(buf);
}
```

#### virtnet_stats

获取网络设备使用情况

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static void virtnet_stats(struct net_device *dev,
			  struct rtnl_link_stats64 *tot)
{
	struct virtnet_info *vi = netdev_priv(dev);
	unsigned int start;
	int i;

	for (i = 0; i < vi->max_queue_pairs; i++) {
		u64 tpackets, tbytes, terrors, rpackets, rbytes, rdrops;
		struct receive_queue *rq = &vi->rq[i];
		struct send_queue *sq = &vi->sq[i];

		do {
			start = u64_stats_fetch_begin(&sq->stats.syncp);
			tpackets = sq->stats.packets;
			tbytes   = sq->stats.bytes;
			terrors  = sq->stats.tx_timeouts;
		} while (u64_stats_fetch_retry(&sq->stats.syncp, start));

		do {
			start = u64_stats_fetch_begin(&rq->stats.syncp);
			rpackets = rq->stats.packets;
			rbytes   = rq->stats.bytes;
			rdrops   = rq->stats.drops;
		} while (u64_stats_fetch_retry(&rq->stats.syncp, start));

		tot->rx_packets += rpackets;
		tot->tx_packets += tpackets;
		tot->rx_bytes   += rbytes;
		tot->tx_bytes   += tbytes;
		tot->rx_dropped += rdrops;
		tot->tx_errors  += terrors;
	}

	tot->tx_dropped = dev->stats.tx_dropped;
	tot->tx_fifo_errors = dev->stats.tx_fifo_errors;
	tot->rx_length_errors = dev->stats.rx_length_errors;
	tot->rx_frame_errors = dev->stats.rx_frame_errors;
}
```

#### virtnet_vlan_rx_add_vid

如果设备支持 VLAN 过滤，则在注册 VLAN id 时调用此函数。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_vlan_rx_add_vid(struct net_device *dev,
				   __be16 proto, u16 vid)
{
	struct virtnet_info *vi = netdev_priv(dev);
	struct scatterlist sg;

	vi->ctrl->vid = cpu_to_virtio16(vi->vdev, vid);
	sg_init_one(&sg, &vi->ctrl->vid, sizeof(vi->ctrl->vid));

	if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_VLAN,
				  VIRTIO_NET_CTRL_VLAN_ADD, &sg))
		dev_warn(&dev->dev, "Failed to add VLAN ID %d.\n", vid);
	return 0;
}
```

#### virtnet_vlan_rx_kill_vid

如果设备支持 `VLAN` 过滤，则在注销 `VLAN id` 时调用此函数。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_vlan_rx_kill_vid(struct net_device *dev,
				    __be16 proto, u16 vid)
{
	struct virtnet_info *vi = netdev_priv(dev);
	struct scatterlist sg;

	vi->ctrl->vid = cpu_to_virtio16(vi->vdev, vid);
	sg_init_one(&sg, &vi->ctrl->vid, sizeof(vi->ctrl->vid));

	if (!virtnet_send_command(vi, VIRTIO_NET_CTRL_VLAN,
				  VIRTIO_NET_CTRL_VLAN_DEL, &sg))
		dev_warn(&dev->dev, "Failed to kill VLAN ID %d.\n", vid);
	return 0;
}
```

#### virtnet_xdp

该函数用于设置或查询网络设备上 `XDP` 相关的状态，管理 `BPF` 卸载。 

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_xdp(struct net_device *dev, struct netdev_bpf *xdp)
{
	switch (xdp->command) {
	case XDP_SETUP_PROG:
		return virtnet_xdp_set(dev, xdp->prog, xdp->extack);
	default:
		return -EINVAL;
	}
}

```

#### virtnet_xdp_xmit

此函数用于提交 `XDP` 数据包在网络设备上传输。 返回成功传输的帧数，丢弃的帧通过 `xdp_return_frame() ` 释放/返回。 返回负数，表示调用 `ndo` 时出现一般错误，表示没有帧被发送，核心调用者将释放所有帧。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_xdp_xmit(struct net_device *dev,
			    int n, struct xdp_frame **frames, u32 flags)
{
	struct virtnet_info *vi = netdev_priv(dev);
	struct receive_queue *rq = vi->rq;
	struct bpf_prog *xdp_prog;
	struct send_queue *sq;
	unsigned int len;
	int packets = 0;
	int bytes = 0;
	int nxmit = 0;
	int kicks = 0;
	void *ptr;
	int ret;
	int i;

	/* Only allow ndo_xdp_xmit if XDP is loaded on dev, as this
	 * indicate XDP resources have been successfully allocated.
	 */
	xdp_prog = rcu_access_pointer(rq->xdp_prog);
	if (!xdp_prog)
		return -ENXIO;

	sq = virtnet_xdp_get_sq(vi);

	if (unlikely(flags & ~XDP_XMIT_FLAGS_MASK)) {
		ret = -EINVAL;
		goto out;
	}

	/* Free up any pending old buffers before queueing new ones. */
	while ((ptr = virtqueue_get_buf(sq->vq, &len)) != NULL) {
		if (likely(is_xdp_frame(ptr))) {
			struct xdp_frame *frame = ptr_to_xdp(ptr);

			bytes += frame->len;
			xdp_return_frame(frame);
		} else {
			struct sk_buff *skb = ptr;

			bytes += skb->len;
			napi_consume_skb(skb, false);
		}
		packets++;
	}

	for (i = 0; i < n; i++) {
		struct xdp_frame *xdpf = frames[i];

		if (__virtnet_xdp_xmit_one(vi, sq, xdpf))
			break;
		nxmit++;
	}
	ret = nxmit;

	if (flags & XDP_XMIT_FLUSH) {
		if (virtqueue_kick_prepare(sq->vq) && virtqueue_notify(sq->vq))
			kicks = 1;
	}
out:
	u64_stats_update_begin(&sq->stats.syncp);
	sq->stats.bytes += bytes;
	sq->stats.packets += packets;
	sq->stats.xdp_tx += n;
	sq->stats.xdp_tx_drops += n - nxmit;
	sq->stats.kicks += kicks;
	u64_stats_update_end(&sq->stats.syncp);

	virtnet_xdp_put_sq(vi, sq);
	return ret;
}
```

#### passthru_features_check

确定设备是否能够对给定数据包执行卸载操作。

`源码位置` [/net/core/dev.c](https://github1s.com/torvalds/linux/blob/HEAD/net/core/dev.c)

```c
netdev_features_t passthru_features_check(struct sk_buff *skb,
					  struct net_device *dev,
					  netdev_features_t features)
{
	return features;
}
EXPORT_SYMBOL(passthru_features_check);
```

#### virtnet_get_phys_port_name

用于获取物理端口名称，该端口用于与主机机器连接。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_get_phys_port_name(struct net_device *dev, char *buf,
				      size_t len)
{
	struct virtnet_info *vi = netdev_priv(dev);
	int ret;

	if (!virtio_has_feature(vi->vdev, VIRTIO_NET_F_STANDBY))
		return -EOPNOTSUPP;

	ret = snprintf(buf, len, "sby");
	if (ret >= len)
		return -EOPNOTSUPP;

	return 0;
}
```

#### virtnet_set_features

调用以将设备配置更新。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_set_features(struct net_device *dev,
				netdev_features_t features)
{
	struct virtnet_info *vi = netdev_priv(dev);
	u64 offloads;
	int err;

	if ((dev->features ^ features) & NETIF_F_GRO_HW) {
		if (vi->xdp_enabled)
			return -EBUSY;

		if (features & NETIF_F_GRO_HW)
			offloads = vi->guest_offloads_capable;
		else
			offloads = vi->guest_offloads_capable &
				   ~GUEST_OFFLOAD_GRO_HW_MASK;

		err = virtnet_set_guest_offloads(vi, offloads);
		if (err)
			return err;
		vi->guest_offloads = offloads;
	}

	if ((dev->features ^ features) & NETIF_F_RXHASH) {
		if (features & NETIF_F_RXHASH)
			vi->ctrl->rss.hash_types = vi->rss_hash_types_saved;
		else
			vi->ctrl->rss.hash_types = VIRTIO_NET_HASH_REPORT_NONE;

		if (!virtnet_commit_rss_command(vi))
			return -EINVAL;
	}

	return 0;
}
```

#### virtnet_tx_timeout

发送超时回调函数，当发送超时的时候产生会执行，一般都是网络出问题了导致发送超时。一般可能会重启 MAC 和 PHY，重新开始数据发送等。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static void virtnet_tx_timeout(struct net_device *dev, unsigned int txqueue)
{
	struct virtnet_info *priv = netdev_priv(dev);
	struct send_queue *sq = &priv->sq[txqueue];
	struct netdev_queue *txq = netdev_get_tx_queue(dev, txqueue);

	u64_stats_update_begin(&sq->stats.syncp);
	sq->stats.tx_timeouts++;
	u64_stats_update_end(&sq->stats.syncp);

	netdev_err(dev, "TX timeout on queue: %u, sq: %s, vq: 0x%x, name: %s, %u usecs ago\n",
		   txqueue, sq->name, sq->vq->index, sq->vq->name,
		   jiffies_to_usecs(jiffies - READ_ONCE(txq->trans_start)));
}
```

## struct virtio_driver

`源码位置`[/include/linux/virtio.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/virtio.h)

```c
/**
 * struct virtio_driver - operations for a virtio I/O driver
 * @driver: underlying device driver (populate name and owner).
 * @id_table: the ids serviced by this driver.
 * @feature_table: an array of feature numbers supported by this driver.
 * @feature_table_size: number of entries in the feature table array.
 * @feature_table_legacy: same as feature_table but when working in legacy mode.
 * @feature_table_size_legacy: number of entries in feature table legacy array.
 * @probe: the function to call when a device is found.  Returns 0 or -errno.
 * @scan: optional function to call after successful probe; intended
 *    for virtio-scsi to invoke a scan.
 * @remove: the function to call when a device is removed.
 * @config_changed: optional function to call when the device configuration
 *    changes; may be called in interrupt context.
 * @freeze: optional function to call during suspend/hibernation.
 * @restore: optional function to call on resume.
 */
struct virtio_driver {
    /* 驱动程序的基类，提供驱动程序的通用功能，包含驱动程序的操作和事件处理函数 */
	struct device_driver driver;
    /* 设备匹配表，用于将驱动与设备匹配 */
	const struct virtio_device_id *id_table;
    /* 记录驱动程序支持的特性，以数组的形式存放于表中 */
	const unsigned int *feature_table;
    /* 特性表条目数 */
	unsigned int feature_table_size;
    /* 与feature_table功能相同，在legacy模式下调用 */
	const unsigned int *feature_table_legacy;
    /* feature_table_legacy表条目数 */
	unsigned int feature_table_size_legacy;
	int (*validate)(struct virtio_device *dev);
    /* 设备发现时调用 */
	int (*probe)(struct virtio_device *dev);
    /* 扫描设备 */
	void (*scan)(struct virtio_device *dev);
    /* 设备删除时调用 */
	void (*remove)(struct virtio_device *dev);
    /* 设备配置发生改变时调用 */
	void (*config_changed)(struct virtio_device *dev);
#ifdef CONFIG_PM
    /* 设备驱动程序冻结（suspend）操作时调用的函数。 */
	int (*freeze)(struct virtio_device *dev);
    /* 设备驱动程序恢复设备状态 */
	int (*restore)(struct virtio_device *dev);
#endif
};
```

### virtio_net_driver

`virtio_driver` 是一个抽象数据类型，它用来记录 virtio 驱动的相关信息及函数接口，用于驱动程序操作系统内的 virtio 设备。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static struct virtio_driver virtio_net_driver = {
	.feature_table = features,
	.feature_table_size = ARRAY_SIZE(features),
	.feature_table_legacy = features_legacy,
	.feature_table_size_legacy = ARRAY_SIZE(features_legacy),
	.driver.name =	KBUILD_MODNAME,
	.driver.owner =	THIS_MODULE,
	.id_table =	id_table,
	.validate =	virtnet_validate,
	.probe =	virtnet_probe,
	.remove =	virtnet_remove,
	.config_changed = virtnet_config_changed,
#ifdef CONFIG_PM_SLEEP
	.freeze =	virtnet_freeze,
	.restore =	virtnet_restore,
#endif
};
```

#### virtnet_validate

`virtnet_validate` 函数会检查传入的网络包的长度是否合法，并检查包头是否有效。这个函数在 `virtio-net` 驱动的接收流程中被调用，以确保接收到的网络包是有效的，可以被处理。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_validate(struct virtio_device *vdev)
{
	if (!vdev->config->get) {
		dev_err(&vdev->dev, "%s failure: config access disabled\n",
			__func__);
		return -EINVAL;
	}

	if (!virtnet_validate_features(vdev))
		return -EINVAL;

	if (virtio_has_feature(vdev, VIRTIO_NET_F_MTU)) {
		int mtu = virtio_cread16(vdev,
					 offsetof(struct virtio_net_config,
						  mtu));
		if (mtu < MIN_MTU)
			__virtio_clear_bit(vdev, VIRTIO_NET_F_MTU);
	}

	return 0;
}
```

#### virtnet_probe

初始化 virtio 网络设备。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int virtnet_probe(struct virtio_device *vdev)
{
	int i, err = -ENOMEM;
	struct net_device *dev;
	struct virtnet_info *vi;
	u16 max_queue_pairs;
	int mtu = 0;

	/* Find if host supports multiqueue/rss virtio_net device */
	max_queue_pairs = 1;
	if (virtio_has_feature(vdev, VIRTIO_NET_F_MQ) || virtio_has_feature(vdev, VIRTIO_NET_F_RSS))
		max_queue_pairs =
		     virtio_cread16(vdev, offsetof(struct virtio_net_config, max_virtqueue_pairs));

	/* We need at least 2 queue's */
	if (max_queue_pairs < VIRTIO_NET_CTRL_MQ_VQ_PAIRS_MIN ||
	    max_queue_pairs > VIRTIO_NET_CTRL_MQ_VQ_PAIRS_MAX ||
	    !virtio_has_feature(vdev, VIRTIO_NET_F_CTRL_VQ))
		max_queue_pairs = 1;

	/* Allocate ourselves a network device with room for our info */
	dev = alloc_etherdev_mq(sizeof(struct virtnet_info), max_queue_pairs);
	if (!dev)
		return -ENOMEM;

	/* Set up network device as normal. */
	dev->priv_flags |= IFF_UNICAST_FLT | IFF_LIVE_ADDR_CHANGE |
			   IFF_TX_SKB_NO_LINEAR;
	dev->netdev_ops = &virtnet_netdev;
	dev->features = NETIF_F_HIGHDMA;

	dev->ethtool_ops = &virtnet_ethtool_ops;
	SET_NETDEV_DEV(dev, &vdev->dev);

	/* Do we support "hardware" checksums? */
	if (virtio_has_feature(vdev, VIRTIO_NET_F_CSUM)) {
		/* This opens up the world of extra features. */
		dev->hw_features |= NETIF_F_HW_CSUM | NETIF_F_SG;
		if (csum)
			dev->features |= NETIF_F_HW_CSUM | NETIF_F_SG;

		if (virtio_has_feature(vdev, VIRTIO_NET_F_GSO)) {
			dev->hw_features |= NETIF_F_TSO
				| NETIF_F_TSO_ECN | NETIF_F_TSO6;
		}
		/* Individual feature bits: what can host handle? */
		if (virtio_has_feature(vdev, VIRTIO_NET_F_HOST_TSO4))
			dev->hw_features |= NETIF_F_TSO;
		if (virtio_has_feature(vdev, VIRTIO_NET_F_HOST_TSO6))
			dev->hw_features |= NETIF_F_TSO6;
		if (virtio_has_feature(vdev, VIRTIO_NET_F_HOST_ECN))
			dev->hw_features |= NETIF_F_TSO_ECN;
		if (virtio_has_feature(vdev, VIRTIO_NET_F_HOST_USO))
			dev->hw_features |= NETIF_F_GSO_UDP_L4;

		dev->features |= NETIF_F_GSO_ROBUST;

		if (gso)
			dev->features |= dev->hw_features & NETIF_F_ALL_TSO;
		/* (!csum && gso) case will be fixed by register_netdev() */
	}
	if (virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_CSUM))
		dev->features |= NETIF_F_RXCSUM;
	if (virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_TSO4) ||
	    virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_TSO6))
		dev->features |= NETIF_F_GRO_HW;
	if (virtio_has_feature(vdev, VIRTIO_NET_F_CTRL_GUEST_OFFLOADS))
		dev->hw_features |= NETIF_F_GRO_HW;

	dev->vlan_features = dev->features;

	/* MTU range: 68 - 65535 */
	dev->min_mtu = MIN_MTU;
	dev->max_mtu = MAX_MTU;

	/* Configuration may specify what MAC to use.  Otherwise random. */
	if (virtio_has_feature(vdev, VIRTIO_NET_F_MAC)) {
		u8 addr[ETH_ALEN];

		virtio_cread_bytes(vdev,
				   offsetof(struct virtio_net_config, mac),
				   addr, ETH_ALEN);
		eth_hw_addr_set(dev, addr);
	} else {
		eth_hw_addr_random(dev);
	}

	/* Set up our device-specific information */
	vi = netdev_priv(dev);
	vi->dev = dev;
	vi->vdev = vdev;
	vdev->priv = vi;

	INIT_WORK(&vi->config_work, virtnet_config_changed_work);
	spin_lock_init(&vi->refill_lock);

	if (virtio_has_feature(vdev, VIRTIO_NET_F_MRG_RXBUF))
		vi->mergeable_rx_bufs = true;

	if (virtio_has_feature(vi->vdev, VIRTIO_NET_F_NOTF_COAL)) {
		vi->rx_usecs = 0;
		vi->tx_usecs = 0;
		vi->tx_max_packets = 0;
		vi->rx_max_packets = 0;
	}

	if (virtio_has_feature(vdev, VIRTIO_NET_F_HASH_REPORT))
		vi->has_rss_hash_report = true;

	if (virtio_has_feature(vdev, VIRTIO_NET_F_RSS))
		vi->has_rss = true;

	if (vi->has_rss || vi->has_rss_hash_report) {
		vi->rss_indir_table_size =
			virtio_cread16(vdev, offsetof(struct virtio_net_config,
				rss_max_indirection_table_length));
		vi->rss_key_size =
			virtio_cread8(vdev, offsetof(struct virtio_net_config, rss_max_key_size));

		vi->rss_hash_types_supported =
		    virtio_cread32(vdev, offsetof(struct virtio_net_config, supported_hash_types));
		vi->rss_hash_types_supported &=
				~(VIRTIO_NET_RSS_HASH_TYPE_IP_EX |
				  VIRTIO_NET_RSS_HASH_TYPE_TCP_EX |
				  VIRTIO_NET_RSS_HASH_TYPE_UDP_EX);

		dev->hw_features |= NETIF_F_RXHASH;
	}

	if (vi->has_rss_hash_report)
		vi->hdr_len = sizeof(struct virtio_net_hdr_v1_hash);
	else if (virtio_has_feature(vdev, VIRTIO_NET_F_MRG_RXBUF) ||
		 virtio_has_feature(vdev, VIRTIO_F_VERSION_1))
		vi->hdr_len = sizeof(struct virtio_net_hdr_mrg_rxbuf);
	else
		vi->hdr_len = sizeof(struct virtio_net_hdr);

	if (virtio_has_feature(vdev, VIRTIO_F_ANY_LAYOUT) ||
	    virtio_has_feature(vdev, VIRTIO_F_VERSION_1))
		vi->any_header_sg = true;

	if (virtio_has_feature(vdev, VIRTIO_NET_F_CTRL_VQ))
		vi->has_cvq = true;

	if (virtio_has_feature(vdev, VIRTIO_NET_F_MTU)) {
		mtu = virtio_cread16(vdev,
				     offsetof(struct virtio_net_config,
					      mtu));
		if (mtu < dev->min_mtu) {
			/* Should never trigger: MTU was previously validated
			 * in virtnet_validate.
			 */
			dev_err(&vdev->dev,
				"device MTU appears to have changed it is now %d < %d",
				mtu, dev->min_mtu);
			err = -EINVAL;
			goto free;
		}

		dev->mtu = mtu;
		dev->max_mtu = mtu;
	}

	virtnet_set_big_packets(vi, mtu);

	if (vi->any_header_sg)
		dev->needed_headroom = vi->hdr_len;

	/* Enable multiqueue by default */
	if (num_online_cpus() >= max_queue_pairs)
		vi->curr_queue_pairs = max_queue_pairs;
	else
		vi->curr_queue_pairs = num_online_cpus();
	vi->max_queue_pairs = max_queue_pairs;

	/* Allocate/initialize the rx/tx queues, and invoke find_vqs */
    /* 初始化 virtio-net 设备的虚拟队列 */
	err = init_vqs(vi);
	if (err)
		goto free;

#ifdef CONFIG_SYSFS
	if (vi->mergeable_rx_bufs)
		dev->sysfs_rx_queue_group = &virtio_net_mrg_rx_group;
#endif
	netif_set_real_num_tx_queues(dev, vi->curr_queue_pairs);
	netif_set_real_num_rx_queues(dev, vi->curr_queue_pairs);

	virtnet_init_settings(dev);

	if (virtio_has_feature(vdev, VIRTIO_NET_F_STANDBY)) {
		vi->failover = net_failover_create(vi->dev);
		if (IS_ERR(vi->failover)) {
			err = PTR_ERR(vi->failover);
			goto free_vqs;
		}
	}

	if (vi->has_rss || vi->has_rss_hash_report)
		virtnet_init_default_rss(vi);

	/* serialize netdev register + virtio_device_ready() with ndo_open() */
	rtnl_lock();

	err = register_netdevice(dev);
	if (err) {
		pr_debug("virtio_net: registering device failed\n");
		rtnl_unlock();
		goto free_failover;
	}
	
    /* 通知后端 VM 驱动已经 ready */
	virtio_device_ready(vdev);

	rtnl_unlock();

	err = virtnet_cpu_notif_add(vi);
	if (err) {
		pr_debug("virtio_net: registering cpu notifier failed\n");
		goto free_unregister_netdev;
	}

	virtnet_set_queues(vi, vi->curr_queue_pairs);

	/* Assume link up if device can't report link status,
	   otherwise get link status from config. */
	netif_carrier_off(dev);
	if (virtio_has_feature(vi->vdev, VIRTIO_NET_F_STATUS)) {
		schedule_work(&vi->config_work);
	} else {
		vi->status = VIRTIO_NET_S_LINK_UP;
		virtnet_update_settings(vi);
		netif_carrier_on(dev);
	}

	for (i = 0; i < ARRAY_SIZE(guest_offloads); i++)
		if (virtio_has_feature(vi->vdev, guest_offloads[i]))
			set_bit(guest_offloads[i], &vi->guest_offloads);
	vi->guest_offloads_capable = vi->guest_offloads;

	pr_debug("virtnet: registered device %s with %d RX and TX vq's\n",
		 dev->name, max_queue_pairs);

	return 0;

free_unregister_netdev:
	unregister_netdev(dev);
free_failover:
	net_failover_destroy(vi->failover);
free_vqs:
	virtio_reset_device(vdev);
	cancel_delayed_work_sync(&vi->refill);
	free_receive_page_frags(vi);
	virtnet_del_vqs(vi);
free:
	free_netdev(dev);
	return err;
}
```

#### virtnet_remove

删除设备时调用，主要完成删除网络设备、释放相关资源等操作。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static void virtnet_remove(struct virtio_device *vdev)
{
	struct virtnet_info *vi = vdev->priv;

    /* 删除设备对于的CPU通知 */
	virtnet_cpu_notif_remove(vi);

	/* Make sure no work handler is accessing the device. */
    /* 确保 vi->config_work 工作不在访问设备 */
	flush_work(&vi->config_work);

	unregister_netdev(vi->dev);

	net_failover_destroy(vi->failover);

	remove_vq_common(vi);
	
    /* 释放设备 */
	free_netdev(vi->dev);
}
```



## virtnet_ethtool_ops

`ethtool` 是一个用于配置和查询以太网设备的工具，它提供了一组命令行工具和一个用于编写和调试网络驱动程序的库。库中提供很多网络配置的操作，这些操作可以通过 `ethtool` 命令行工具或库函数来访问。例如，要查询网络链接状态，可以使用 `ethtool -k` 命令。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static const struct ethtool_ops virtnet_ethtool_ops = {
	.supported_coalesce_params = ETHTOOL_COALESCE_MAX_FRAMES |
		ETHTOOL_COALESCE_USECS,
	.get_drvinfo = virtnet_get_drvinfo,
	.get_link = ethtool_op_get_link,
	.get_ringparam = virtnet_get_ringparam,
	.set_ringparam = virtnet_set_ringparam,
	.get_strings = virtnet_get_strings,
	.get_sset_count = virtnet_get_sset_count,
	.get_ethtool_stats = virtnet_get_ethtool_stats,
	.set_channels = virtnet_set_channels,
	.get_channels = virtnet_get_channels,
	.get_ts_info = ethtool_op_get_ts_info,
	.get_link_ksettings = virtnet_get_link_ksettings,
	.set_link_ksettings = virtnet_set_link_ksettings,
	.set_coalesce = virtnet_set_coalesce,
	.get_coalesce = virtnet_get_coalesce,
	.get_rxfh_key_size = virtnet_get_rxfh_key_size,
	.get_rxfh_indir_size = virtnet_get_rxfh_indir_size,
	.get_rxfh = virtnet_get_rxfh,
	.set_rxfh = virtnet_set_rxfh,
	.get_rxnfc = virtnet_get_rxnfc,
	.set_rxnfc = virtnet_set_rxnfc,
};
```







# 参考

[virtio-net 实现机制【二】（图文并茂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545258186)

[Linux 网络子系统 - 陈富林 - 博客园 (cnblogs.com)](https://www.cnblogs.com/chenfulin5/p/6073755.html)
