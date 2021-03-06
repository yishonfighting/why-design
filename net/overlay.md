### Overlay 网络

并不是突然说到overlay啊，最近在看kubernetes相关的内容，后面会在收集一些overlay在kubernetes应用的知识。

当然overlay也并不是kubernetes的专有名词，其实延展网络是一门计算机网络比较旧的技术了，具体就是指构建再另外一个网络上的计算机网络，也就是
一种网络虚拟化技术，近年在云计算虚拟化技术就是网络虚拟化技术的应用。

跳回来，因为overlay网络是建立在另外一个计算机网络上的虚拟网络，那么依赖的网络就是underlay网络。underlay网络是专门用来承载用户IP流量的基础架构层，两者的关系类似物理机和虚拟机，对应的，underlay网络和虚拟机是真正存在的实体，它其实就是真实存在的网络设备和计算设备，而overlay网络和虚拟机都是依托在下层尸体使用软件虚拟出来的层级。

在组件overlay网络的时候，我们通常会使用虚拟局域网拓展技术（virtual extensible LAN， VxLAN）,VxLAN会使用虚拟隧道端点设备对服务器发出和收到的数据包进行二次封装和解封。

通俗点说，在一个overlay网络中，两个服务器之间可以通过V下LAN的VTEP互相连接，并获得网络中的MAC地址、IP地址等。由此，在数据包传输的过程中，通信的双方都不知道底层网络怎么做的转换，它们只会认为两者可以通过二层网络互相访问，但实际经过了三层IP网络的中转。当然，VxLAN只是其中一个实现方式，也有其他的实现，不过都大同小异。

优缺点其实看完上面的内容，都挺明显的，overlay虽然可以利用底层网络在多数据中心之间组成二层网络，但是它的封包拆包过程也会带来额外的开销，那什么集群中还需要overlay网络呢？

#### 频繁的虚拟机迁移

越来越多的计算任务跑在虚拟机上，虚拟机迁移是将虚拟机从一个物理硬件设备移到另一个设备的过程，比如日常的更新维护，集群中需要进行大规模虚拟机迁移是非常常见的事情，迁移可以带来资源利用率的提高。

当虚拟机所在的宿主机因为维护或者其他原因宕机的时候，当前实例就要迁移到其他宿主机上，以保证业务不中断，那么我们在这个过程需要做的就是保证IP地址不变，恰恰，overlay是在网络层实现二层网络，所以多个物理机之间只要网络层可达就可以组建成虚拟的局域网，就算是虚拟机/容器进行了迁移，它们仍然处于同一个二层网络，自然IP地址也不需要改变。这些处理对于集群内部的主机来说，完全不清楚，也不需要关心底层的网络架构，它们只知道不同虚拟机之间是可以连接的。

#### 大规模迁移带来的压力

二层网络的通信需要依赖MAC地址。目前Kubernetes官方支持的最大集群为5000个节点，假设每个节点都只包含一个容器，这其实对内部的网络设备来说其实没有太大的压力。但是现实是，5000节点的集群往往包含几万甚至几十万个容器，当某个容器向集群发送ARP请求的时候，集群中的全部容器都会收到ARP请求，这个时候会带来极高的网络负载。

还是以V下LAN实现的overlay网络为例子，网络会将虚拟机发送的数据重新封装成IP数据包，这样网络只需要知道不同VTEP的MAC地址，由此将MAC地址表项中的几十万条数据拆分降低到几千条，ARP请求也只会在集群中的VTEP之间扩散，远端的VTEP将数据拆包后也仅仅只会在本地广播，不会影响其他的VTEP，降低了核心网络设备的压力，不过对集群中的网络设备仍然会有较高的要求。


#### 更高的网络隔离需求

传统的网络隔离技术VLAN只能建立4096个虚拟网络，公有云以及大规模的虚拟化集群需要更多的虚拟网络才能满足网络隔离的需求。

大规模的数据中心往往都会对外提供云计算服务，同一个物理集群可能会被拆分成多个小块分配给不同的租户（Tenant），因为二层网络的数据帧可能会进行广播，所以出于安全的考虑这些不同的租户之间需要进行网络隔离，避免租户之间的流量互相影响甚至恶意攻击。传统的网络隔离会使用虚拟局域网技术（Virtual LAN、VLAN），VLAN会使用12比特表示虚拟网络ID，虚拟网络的上限是4096个。但是4096这个数量对于大规模的数据中心来说是远远不够的，还是以VxLAN为例，会使用24比特的VNI表示虚拟网络个数，这个时候总共就可以表示16777216个虚拟网络。但是这里有个误区，并不是因为支持更多的虚拟网络才要选择overlay网络，这只是充分条件而已，使用两个VLAN ID组成的24比特同样可以解决这个问题。


#### 结语

通过Overlay网络这一个中间层，我们可以解决虚拟机的迁移问题、降低二层核心网络设备的压力并提供更大规模的虚拟网络数量。需要注意的是，Overlay 网络只是一种在物理网络上的虚拟网络，使用该技术并不能直接解决集群中的规模性等问题，而 VxLAN 也不是组建 Overlay 网络的唯一方法，在不同场景中我们可以考虑使用不同的技术，例如：NVGRE、GRE等。

- 使用VxLAN构成二层网络中，虚拟机在不同集群、不同可用区和不同数据中心迁移后，仍然可以保证二层网络的可达性，这能够帮助我们保证线上业务的可用性、提升集群的资源利用率、容忍虚拟机和节点的故障；
- 集群中虚拟机的规模可能是物理机是几十倍，与物理机构成的传统集群相比，虚拟机构成的集群包含的 MAC 地址数量可能多一两个数量级，网络设备很难承担如此大规模的二层网络请求，Overlay网络通过IP封包和控制平面可以减少集群中的MAC地址表项和ARP请求；
- VxLAN 的协议头使用24位的VNI表示虚拟网络，总共可以表示1600万的虚拟网络，我们可以为不同的虚拟网络单独分配网络带宽，满足多租户的网络隔离需求；







