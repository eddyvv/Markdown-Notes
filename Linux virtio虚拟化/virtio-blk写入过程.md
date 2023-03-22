# virtio-blk存储虚拟化写入流程



![img](image/virtio-blk%E5%86%99%E5%85%A5%E8%BF%87%E7%A8%8B/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNTg4NzMw,size_16,color_FFFFFF,t_70.jpeg)

## 前端驱动

```c
virtio_queue_rq
    -->virtblk_prep_rq
    	-->virtblk_map_data
    		-->blk_rq_map_sg
    			-->__blk_rq_map_sg
    -->virtblk_add_req
    	-->sg_init_one
    	-->virtqueue_add_sgs
    		-->virtqueue_add	/* 向queue中添加数据 */
    -->virtqueue_kick_prepare
    -->virtqueue_notify	/* 通知后端驱动 */
```



## 后端驱动

```c
vm-exit
	-->virtio_ioport_write
    	-->virtio_queue_notify
			-->vq->handle_output(vdev, vq)	/* 对应相应模块handle_output函数 */
```















