# virtio-net 后端驱动

Virtio-Net设备，结构体及拓扑组织关系如下图

![img](image/virtio-net%E5%90%8E%E7%AB%AF%E9%A9%B1%E5%8A%A8/ea964ab6aecb4a5d2cc89a590f5b361c.png)

> 图片引用自[Linux虚拟化KVM-Qemu分析之Virtio设备-51CTO.COM](https://www.51cto.com/article/645696.html)

## virtio_net_info

`virtio-net` 这种类有多层的继承关系，`TYPE_VIRTIO_NET` 的父类是 `TYPE_VIRTIO_DEVICE` ， `TYPE_VIRTIO_DEVICE` 的父类是 `TYPE_DEVICE` ， `TYPE_DEVICE` 的父类是 `TYPE_OBJECT` ，此处继承关系结束。

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(DeviceState),
    .instance_init = device_initfn,
    .instance_post_init = device_post_init,
    .instance_finalize = device_finalize,
    .class_base_init = device_class_base_init,
    .class_init = device_class_init,
    .abstract = true,
    .class_size = sizeof(DeviceClass),
    .interfaces = (InterfaceInfo[]) {
        { TYPE_VMSTATE_IF },
        { TYPE_RESETTABLE_INTERFACE },
        { }
    }
};

static const TypeInfo virtio_device_info = {
    .name = TYPE_VIRTIO_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(VirtIODevice),
    .class_init = virtio_device_class_init,
    .instance_finalize = virtio_device_instance_finalize,
    .abstract = true,
    .class_size = sizeof(VirtioDeviceClass),
};

static const TypeInfo virtio_net_info = {
    .name = TYPE_VIRTIO_NET,
    .parent = TYPE_VIRTIO_DEVICE,
    .instance_size = sizeof(VirtIONet),
    .instance_init = virtio_net_instance_init,
    .class_init = virtio_net_class_init,
};

static void virtio_register_types(void)
{
    type_register_static(&virtio_net_info);
}

type_init(virtio_register_types)
```

其继承关系如下：

```c
TYPE_OBJECT
->	TYPE_DEVICE
	->	TYPE_VIRTIO_DEVICE
		->	TYPE_VIRTIO_NET
```

在 `virtio_net_info` 中 `instance_init` 用于完成将 `xxxClass` 初始化为实例，`class_init` 用于从`TypeImpl` 生成 `xxxClass`。

## virtio_net_instance_init

`virtio_net_instance_init` 完成 `virtio-net` 设备的实例初始化。

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_instance_init(Object *obj)
{
    VirtIONet *n = VIRTIO_NET(obj);

    /*
     * The default config_size is sizeof(struct virtio_net_config).
     * Can be overriden with virtio_net_set_config_size.
     */
    n->config_size = sizeof(struct virtio_net_config);
    device_add_bootindex_property(obj, &n->nic_conf.bootindex,
                                  "bootindex", "/ethernet-phy@0",
                                  DEVICE(n));

    ebpf_rss_init(&n->ebpf_rss);
}
```

## virtio_net_class_init

`virtio_net_class_init` 是 `virtio-net` 设备的类初始化函数，主要完成 `VirtioDeviceClass` 成员函数的赋值。在网卡虚拟化过程中，参数只需要指定 `PCI` 代理设备即可，也就是 `-device virtio-net-pci, netdev=tap0` ，从而会调用到 `virtio_net_pci_class_init` 函数。

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_CLASS(klass);

    /* 设置设备类的属性 */
    device_class_set_props(dc, virtio_net_properties);
    /* 描述虚拟网卡状态信息 */
    dc->vmsd = &vmstate_virtio_net;
    set_bit(DEVICE_CATEGORY_NETWORK, dc->categories);
    vdc->realize = virtio_net_device_realize;
    vdc->unrealize = virtio_net_device_unrealize;
    vdc->get_config = virtio_net_get_config;
    vdc->set_config = virtio_net_set_config;
    vdc->get_features = virtio_net_get_features;
    vdc->set_features = virtio_net_set_features;
    vdc->bad_features = virtio_net_bad_features;
    vdc->reset = virtio_net_reset;
    vdc->queue_reset = virtio_net_queue_reset;
    vdc->queue_enable = virtio_net_queue_enable;
    vdc->set_status = virtio_net_set_status;
    vdc->guest_notifier_mask = virtio_net_guest_notifier_mask;
    vdc->guest_notifier_pending = virtio_net_guest_notifier_pending;
    vdc->legacy_features |= (0x1 << VIRTIO_NET_F_GSO);
    vdc->post_load = virtio_net_post_load_virtio;
    vdc->vmsd = &vmstate_virtio_net_device;
    vdc->primary_unplug_pending = primary_unplug_pending;
    vdc->get_vhost = virtio_net_get_vhost;
}
```

`TYPE_VIRTIO_NET` 层的 `class_init` 函数是 `virtio_net_class_init` ，它定义了 `DeviceClass` 的 `realize` 函数为 `virtio_net_device_realize` 。

### virtio_net_device_realize

当虚拟机启动时，虚拟网络设备会被创建，并通过调用 `virtio_net_device_realize` 函数来将其添加到宿主机的网络子系统中。

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_device_realize(DeviceState *dev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIONet *n = VIRTIO_NET(dev);
    NetClientState *nc;
    int i;

    if (n->net_conf.mtu) {
        n->host_features |= (1ULL << VIRTIO_NET_F_MTU);
    }

    if (n->net_conf.duplex_str) {
        if (strncmp(n->net_conf.duplex_str, "half", 5) == 0) {
            n->net_conf.duplex = DUPLEX_HALF;
        } else if (strncmp(n->net_conf.duplex_str, "full", 5) == 0) {
            n->net_conf.duplex = DUPLEX_FULL;
        } else {
            error_setg(errp, "'duplex' must be 'half' or 'full'");
            return;
        }
        n->host_features |= (1ULL << VIRTIO_NET_F_SPEED_DUPLEX);
    } else {
        n->net_conf.duplex = DUPLEX_UNKNOWN;
    }

    if (n->net_conf.speed < SPEED_UNKNOWN) {
        error_setg(errp, "'speed' must be between 0 and INT_MAX");
        return;
    }
    if (n->net_conf.speed >= 0) {
        n->host_features |= (1ULL << VIRTIO_NET_F_SPEED_DUPLEX);
    }

    if (n->failover) {
        n->primary_listener.hide_device = failover_hide_primary_device;
        qatomic_set(&n->failover_primary_hidden, true);
        device_listener_register(&n->primary_listener);
        n->migration_state.notify = virtio_net_migration_state_notifier;
        add_migration_state_change_notifier(&n->migration_state);
        n->host_features |= (1ULL << VIRTIO_NET_F_STANDBY);
    }

    virtio_net_set_config_size(n, n->host_features);
    /* virtio层，初始化设备 */
    virtio_init(vdev, VIRTIO_ID_NET, n->config_size);

    /*
     * We set a lower limit on RX queue size to what it always was.
     * Guests that want a smaller ring can always resize it without
     * help from us (using virtio 1 and up).
     */
    if (n->net_conf.rx_queue_size < VIRTIO_NET_RX_QUEUE_MIN_SIZE ||
        n->net_conf.rx_queue_size > VIRTQUEUE_MAX_SIZE ||
        !is_power_of_2(n->net_conf.rx_queue_size)) {
        error_setg(errp, "Invalid rx_queue_size (= %" PRIu16 "), "
                   "must be a power of 2 between %d and %d.",
                   n->net_conf.rx_queue_size, VIRTIO_NET_RX_QUEUE_MIN_SIZE,
                   VIRTQUEUE_MAX_SIZE);
        virtio_cleanup(vdev);
        return;
    }

    if (n->net_conf.tx_queue_size < VIRTIO_NET_TX_QUEUE_MIN_SIZE ||
        n->net_conf.tx_queue_size > VIRTQUEUE_MAX_SIZE ||
        !is_power_of_2(n->net_conf.tx_queue_size)) {
        error_setg(errp, "Invalid tx_queue_size (= %" PRIu16 "), "
                   "must be a power of 2 between %d and %d",
                   n->net_conf.tx_queue_size, VIRTIO_NET_TX_QUEUE_MIN_SIZE,
                   VIRTQUEUE_MAX_SIZE);
        virtio_cleanup(vdev);
        return;
    }

    n->max_ncs = MAX(n->nic_conf.peers.queues, 1);

    /*
     * Figure out the datapath queue pairs since the backend could
     * provide control queue via peers as well.
     */
    if (n->nic_conf.peers.queues) {
        for (i = 0; i < n->max_ncs; i++) {
            if (n->nic_conf.peers.ncs[i]->is_datapath) {
                ++n->max_queue_pairs;
            }
        }
    }
    n->max_queue_pairs = MAX(n->max_queue_pairs, 1);

    /* 前端与后端交互数据的队列，最多有VIRTIO_QUEUE_MAX个，这里 
     * n->max_queue_pairs * 2 是因为virtio-net存在两个队列，一个用于接收，一个用于发送 
     */
    if (n->max_queue_pairs * 2 + 1 > VIRTIO_QUEUE_MAX) {
        error_setg(errp, "Invalid number of queue pairs (= %" PRIu32 "), "
                   "must be a positive integer less than %d.",
                   n->max_queue_pairs, (VIRTIO_QUEUE_MAX - 1) / 2);
        virtio_cleanup(vdev);
        return;
    }
    n->vqs = g_new0(VirtIONetQueue, n->max_queue_pairs);
    n->curr_queue_pairs = 1;
    n->tx_timeout = n->net_conf.txtimer;

    if (n->net_conf.tx && strcmp(n->net_conf.tx, "timer")
                       && strcmp(n->net_conf.tx, "bh")) {
        warn_report("virtio-net: "
                    "Unknown option tx=%s, valid options: \"timer\" \"bh\"",
                    n->net_conf.tx);
        error_printf("Defaulting to \"bh\"");
    }

    n->net_conf.tx_queue_size = MIN(virtio_net_max_tx_queue_size(n),
                                    n->net_conf.tx_queue_size);

    /* 初始化队列 */
    for (i = 0; i < n->max_queue_pairs; i++) {
        virtio_net_add_queue(n, i);
    }

    n->ctrl_vq = virtio_add_queue(vdev, 64, virtio_net_handle_ctrl);
    qemu_macaddr_default_if_unset(&n->nic_conf.macaddr);
    memcpy(&n->mac[0], &n->nic_conf.macaddr, sizeof(n->mac));
    n->status = VIRTIO_NET_S_LINK_UP;
    qemu_announce_timer_reset(&n->announce_timer, migrate_announce_params(),
                              QEMU_CLOCK_VIRTUAL,
                              virtio_net_announce_timer, n);
    n->announce_timer.round = 0;

    if (n->netclient_type) {
        /*
         * Happen when virtio_net_set_netclient_name has been called.
         * 创建虚拟机中的网卡
         */
        n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf,
                              n->netclient_type, n->netclient_name, n);
    } else {
        n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf,
                              object_get_typename(OBJECT(dev)), dev->id, n);
    }

    for (i = 0; i < n->max_queue_pairs; i++) {
        n->nic->ncs[i].do_not_pad = true;
    }

    peer_test_vnet_hdr(n);
    if (peer_has_vnet_hdr(n)) {
        for (i = 0; i < n->max_queue_pairs; i++) {
            qemu_using_vnet_hdr(qemu_get_subqueue(n->nic, i)->peer, true);
        }
        n->host_hdr_len = sizeof(struct virtio_net_hdr);
    } else {
        n->host_hdr_len = 0;
    }

    qemu_format_nic_info_str(qemu_get_queue(n->nic), n->nic_conf.macaddr.a);

    n->vqs[0].tx_waiting = 0;
    n->tx_burst = n->net_conf.txburst;
    virtio_net_set_mrg_rx_bufs(n, 0, 0, 0);
    n->promisc = 1; /* for compatibility */

    n->mac_table.macs = g_malloc0(MAC_TABLE_ENTRIES * ETH_ALEN);

    n->vlans = g_malloc0(MAX_VLAN >> 3);

    nc = qemu_get_queue(n->nic);
    nc->rxfilter_notify_enabled = 1;

   if (nc->peer && nc->peer->info->type == NET_CLIENT_DRIVER_VHOST_VDPA) {
        struct virtio_net_config netcfg = {};
        memcpy(&netcfg.mac, &n->nic_conf.macaddr, ETH_ALEN);
        vhost_net_set_config(get_vhost_net(nc->peer),
            (uint8_t *)&netcfg, 0, ETH_ALEN, VHOST_SET_CONFIG_TYPE_MASTER);
    }
    QTAILQ_INIT(&n->rsc_chains);
    n->qdev = dev;

    net_rx_pkt_init(&n->rx_pkt, false);

    if (virtio_has_feature(n->host_features, VIRTIO_NET_F_RSS)) {
        virtio_net_load_ebpf(n);
    }
}
```

#### virtio_net_add_queue

`virtio_net_add_queue` 来初始化队列，可以看出这里面就有发送 `tx_vq` 和接收 `rx_vq` 两个队列，这里注册了中断函数，对于接收数据包，注册为 `virtio_net_handle_rx`，对于发送数据包，注册为 `virtio_net_handle_tx_timer` 和 `virtio_net_handle_tx_bh`。

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_add_queue(VirtIONet *n, int index)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(n);

    n->vqs[index].rx_vq = virtio_add_queue(vdev, n->net_conf.rx_queue_size,
                                           virtio_net_handle_rx);

    if (n->net_conf.tx && !strcmp(n->net_conf.tx, "timer")) {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_timer);
        n->vqs[index].tx_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL,
                                              virtio_net_tx_timer,
                                              &n->vqs[index]);
    } else {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_bh);
        n->vqs[index].tx_bh = qemu_bh_new(virtio_net_tx_bh, &n->vqs[index]);
    }

    n->vqs[index].tx_waiting = 0;
    n->vqs[index].n = n;
}
```

#### qemu_new_nic

`qemu_new_nic` 会创建一个虚拟机里面的网卡。

`源码位置` [/net/net.c](https://github1s.com/qemu/qemu/blob/HEAD/net/net.c)

```c
NICState *qemu_new_nic(NetClientInfo *info,
                       NICConf *conf,
                       const char *model,
                       const char *name,
                       void *opaque)
{
    NetClientState **peers = conf->peers.ncs;
    NICState *nic;
    int i, queues = MAX(1, conf->peers.queues);

    assert(info->type == NET_CLIENT_DRIVER_NIC);
    assert(info->size >= sizeof(NICState));

    nic = g_malloc0(info->size + sizeof(NetClientState) * queues);
    nic->ncs = (void *)nic + info->size;
    nic->conf = conf;
    nic->opaque = opaque;

    for (i = 0; i < queues; i++) {
        qemu_net_client_setup(&nic->ncs[i], info, peers[i], model, name,
                              NULL, true);
        nic->ncs[i].queue_index = i;
    }

    return nic;
}
```

### virtio_net_device_unrealize

`源码位置` [/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_device_unrealize(DeviceState *dev)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIONet *n = VIRTIO_NET(dev);
    int i, max_queue_pairs;

    if (virtio_has_feature(n->host_features, VIRTIO_NET_F_RSS)) {
        virtio_net_unload_ebpf(n);
    }

    /* This will stop vhost backend if appropriate. */
    virtio_net_set_status(vdev, 0);

    g_free(n->netclient_name);
    n->netclient_name = NULL;
    g_free(n->netclient_type);
    n->netclient_type = NULL;

    g_free(n->mac_table.macs);
    g_free(n->vlans);

    if (n->failover) {
        qobject_unref(n->primary_opts);
        device_listener_unregister(&n->primary_listener);
        remove_migration_state_change_notifier(&n->migration_state);
    } else {
        assert(n->primary_opts == NULL);
    }

    max_queue_pairs = n->multiqueue ? n->max_queue_pairs : 1;
    for (i = 0; i < max_queue_pairs; i++) {
        virtio_net_del_queue(n, i);
    }
    /* delete also control vq */
    virtio_del_queue(vdev, max_queue_pairs * 2);
    qemu_announce_timer_del(&n->announce_timer, false);
    g_free(n->vqs);
    qemu_del_nic(n->nic);
    virtio_net_rsc_cleanup(n);
    g_free(n->rss_data.indirections_table);
    net_rx_pkt_uninit(n->rx_pkt);
    virtio_cleanup(vdev);
}
```





# 参考

[网络虚拟化Virtio-net - tycoon3 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dream397/p/14386024.html)

[Linux虚拟化原理笔记_书忆江南的博客-CSDN博客_type_register_internal](https://blog.csdn.net/qq_33588730/article/details/105397879)

[Linux虚拟化KVM-Qemu分析之Virtio设备-51CTO.COM](https://www.51cto.com/article/645696.html)
