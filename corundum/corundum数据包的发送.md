# corundum数据包的发送步骤

对于数据链路层，数据包的发送从函数`ndo_start_xmit()`开始

1. Linux 内核通过 `ndo_start_xmit()` 调用 `mqnic_start_xmit()` （通过 `ndo_start_xmit()` ）并使用 `sk_buff` 进行传输；
2. `mqnic_start_xmit()` ( `mqnic_tx.c` )：驱动程序通过 `skb_get_queue_mapping` 确定目标传输队列；
3. `mqnic_start_xmit()` ( `mqnic_tx.c` )：如果需要，驱动程序将 `sk_buff` 标记为时间戳；
4. `mqnic_start_xmit()` ( `mqnic_tx.c` )：驱动程序生成硬件IP校验和命令并将其写入描述符；
5. `mqnic_map_skb()` ( `mqnic_tx.c` )：驱动程序将对 `sk_buff` 的引用写入 `ring->tx_info`；
6. `mqnic_map_skb()` ( `mqnic_tx.c` )：驱动程序为 `sk_buff` ( `skb_frag_dma_map()` / `dma_map_single()` ) 生成 DMA 映射并构建描述符；
7. `mqnic_start_xmit()` ( `mqnic_tx.c` )：驱动程序通过递增生产者指针的本地副本将数据包排入队列；
8. `mqnic_start_xmit()` ( `mqnic_tx.c` )：在一批数据包结束时，驱动程序通过 MMIO 将更新的生产者指针写入 NIC；

