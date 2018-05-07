---
layout: post
title: topology-manager总结
tags:
- ODL控制器
- SDN开发
categories: SDN开发
description: 鉴于网上对于sdn开发相关的资料较少又乱的现状，从这篇文章开始，我将陆续分享我在sdn开发过程中的经验。
---

topology-manager模块也是作为openflowplugin的应用层程序（APP），位于openflowplugin-release-lithium-sr3文件夹的applications当中，负责处理operational数据库下的network-topology:network-topology数据节点（datastore数据库）的增删改查，例如odl控制器发现加一台主机host，新加主机与交换机的link链接等。显示拓扑的前端需要从该数据节点上获取主机或者交换机节点数据才能绘制网络拓扑图，构成拓扑图来源有两方面，一方面是通过LLDP发现的switch设备以及相关link连接，另一外面是通过L2switch的hosttracker模块发现的挂在switch上的host主机以及相关连接。

#### 1 YANG数据模型

network-topology:network-topology数据节点是直接使用了外部ietf-topology定义的yang文件，主要的数据树结构为：

```xml
container nodes {
        description "The root container of all nodes.";
        list node {
            key "id";
            ext:context-instance "node-context";
            description "A list of nodes (as defined by the 'grouping node').";
            uses node; //this refers to the 'grouping node' defined above.
        }
    }
    grouping node {
        description "Describes the contents of a generic node -
                     essentially an ID and a list of node-connectors.
                     Acts as an augmentation point where other yang files
                      can add additional information.";

        leaf id {
            type node-id;
            description "The unique identifier for the node.";
        }
        list "node-connector" {
            key "id";
            description "A list of node connectors that belong this node.";
            ext:context-instance "node-connector-context";
            uses node-connector;
        }
}
    grouping node-connector {
        description "Describes a generic node connector which consists of an ID.
                     Acts as an augmentation point where other yang files can
                      add additional information.";
        leaf id {
            type node-connector-id;
            description "The unique identifier for the node-connector.";
        }
}
```

#### 2相关依赖

在topology-manager的处理流程需要依赖两条主线，第一是在applications文件夹下的topology-lldp-discovery模块会处理交换机与交换机的link连接(大概流程是lldp-speaker发送lldp报文探测新加入的设备是否为openflow交换机，如果是交换机则由topology-lldp-discovery处理并发出link发现的通告)，因此topology-lldp-discovery模块会发出两类通告FlowTopologyDiscoveryListener：

onLinkDiscovered    //链接添加

onLinkRemoved     //链接移除

第二是topology-manager需要监听Nodes/Node/NodeConnector/FlowCapableNodeConnector和Nodes/Node/NodeConnector/FlowCapableNode数据的变化，此数据就是在inventory-manager模块添加的交换机信息节点，所以当inventory-manager模块新加或者移除一台交换机节点时，会触发Nodes/Node/NodeConnector/FlowCapableNodeConnector

以及Nodes/Node/NodeConnector/FlowCapableNode数据节点的变化，进而引起topology-manager模块进入

processAddedTerminationPoints

processUpdatedTerminationPoints

processRemovedTerminationPoints

processAddedNode

processRemovedNode

处理流程。

#### 3代码流程

topology-manager的入口在TopologyLldpDiscoveryImplModule.java文件的createInstance()函数，进入该函数后new出FlowCapableTopologyProvider对象，然后注册FlowTopologyDiscoveryListener通告的接收器FlowCapableTopologyExporter，同时设置监听Nodes/Node/NodeConnector/FlowCapableNodeConnector

以及Nodes/Node/NodeConnector/FlowCapableNode的数据变化，

先分析FlowTopologyDiscoveryListener通告的接收器FlowCapableTopologyExporter。

###### 3.1 onLinkDiscovered事件

当控制器通过LLDP发现新加的设备与现有设备存在一条链路时（交换机与交换机之间的链路），会触发onLinkDiscovered函数的调用，将该link保存在数据库当中。

```xml
public synchronized void onNodeUpdated(final NodeUpdated node) {
        //判断该节点是否是需要delete的节点
        if (deletedNodeCache.getIfPresent(node.getNodeRef()) != null){
            deletedNodeCache.invalidate(node.getNodeRef());
        }
        //获取需要写入的数据树节点位置
        final FlowCapableNodeUpdated flowNode = node.getAugmentation(FlowCapableNodeUpdated.class);
        if (flowNode == null) {
            return;
        }
        LOG.debug("Node updated notification received,{}", node.getNodeRef().getValue());
        manager.enqueue(new InventoryOperation() {
            @Override
            public void applyOperation(ReadWriteTransaction tx) {
                final NodeRef ref = node.getNodeRef();
                @SuppressWarnings("unchecked")
                //组装node数据
                InstanceIdentifierBuilder<Node> builder = ((InstanceIdentifier<Node>) ref.getValue()).builder();
                InstanceIdentifierBuilder<FlowCapableNode> augmentation = builder.augmentation(FlowCapableNode.class);
                final InstanceIdentifier<FlowCapableNode> path = augmentation.build();
                CheckedFuture<Optional<FlowCapableNode>, ?> readFuture = tx.read(LogicalDatastoreType.OPERATIONAL, path);
                Futures.addCallback(readFuture, new FutureCallback<Optional<FlowCapableNode>>() {
                    @Override
                    public void onSuccess(Optional<FlowCapableNode> optional) {
                        enqueueWriteNodeDataTx(node, flowNode, path);
                        if (!optional.isPresent()) {
                            enqueuePutTable0Tx(ref);
                        }
                    }

                    @Override
                    public void onFailure(Throwable throwable) {
                        LOG.debug(String.format("Can't retrieve node data for node %s. Writing node data with table0.", node));
                        enqueueWriteNodeDataTx(node, flowNode, path);
                        enqueuePutTable0Tx(ref);
                    }
                });
            }
        });
    }

```

###### 3.2 onLinkRemoved事件

当上述链接断开之后，就会触发onLinkRemoved函数的调用，将之前保存在数据库当中的link信息删除。

```xml
public synchronized void onNodeRemoved(final NodeRemoved node) {
        if(deletedNodeCache.getIfPresent(node.getNodeRef()) == null){
            deletedNodeCache.put(node.getNodeRef(), Boolean.TRUE);
        } else {
            //its been noted that creating an operation for already removed node, fails
            // the entire transaction chain, there by failing deserving removals
            LOG.debug("Already received notification to remove node, {} - Ignored",
                    node.getNodeRef().getValue());
            return;
        }
        LOG.debug("Node removed notification received, {}", node.getNodeRef().getValue());
        manager.enqueue(new InventoryOperation() {
            @Override
            public void applyOperation(final ReadWriteTransaction tx) {
                final NodeRef ref = node.getNodeRef();
                LOG.debug("removing node : {}", ref.getValue());
                tx.delete(LogicalDatastoreType.OPERATIONAL, ref.getValue());
            }
        });
    }

```

###### 3.3 FlowCapableNode数据变化事件

再来分析监听Nodes/Node/NodeConnector/FlowCapableNode的数据变化的NodeChangeListenerImpl，当Nodes/Node/NodeConnector/FlowCapableNode的数据发生变化（inventory-manager当中新加/删除一台交换机节点），会触发NodeChangeListenerImpl的onDataChanged函数调用，在该函数当中调用了以下两个函数

processAddedNode(change.getCreatedData());

processRemovedNode(change.getRemovedPaths());

两个函数在自己内部判断是否是节点添加/删除事件，最终将交换机节点信息写入network-topology数据树当中。

```xml
public synchronized void onNodeConnectorRemoved(final NodeConnectorRemoved connector) {
        if(deletedNodeConnectorCache.getIfPresent(connector.getNodeConnectorRef()) == null){
            deletedNodeConnectorCache.put(connector.getNodeConnectorRef(), Boolean.TRUE);
        } else {
            //its been noted that creating an operation for already removed node-connectors, fails
            // the entire transaction chain, there by failing deserving removals
            LOG.debug("Already received notification to remove nodeConnector, {} - Ignored",
                    connector.getNodeConnectorRef().getValue());
            return;
        }
        LOG.debug("Node connector removed notification received, {}", connector.getNodeConnectorRef().getValue());
        manager.enqueue(new InventoryOperation() {
            @Override
            public void applyOperation(final ReadWriteTransaction tx) {
                final NodeConnectorRef ref = connector.getNodeConnectorRef();
                LOG.debug("removing node connector {} ", ref.getValue());
                tx.delete(LogicalDatastoreType.OPERATIONAL, ref.getValue());
            }
        });
}

```

###### 3.4 FlowCapableNodeConnector数据变化事件

最后分析Nodes/Node/NodeConnector/FlowCapableNodeConnector的数据监听类TerminationPointChangeListenerImpl，当Nodes/Node/NodeConnector/FlowCapableNodeConnector数据发生变化时，会触发onDataChanged函数调用

processAddedTerminationPoints(change.getCreatedData());

processUpdatedTerminationPoints(change.getUpdatedData());

processRemovedTerminationPoints(change.getRemovedPaths());

类似的，上述3个函数也是内部判断是否是数据新加/更新/删除事件，假如是新加事件，则调用createData函数将TerminationPoint写入network-topology数据树。

```xml
public synchronized void onNodeConnectorUpdated(final NodeConnectorUpdated connector) {
        if (deletedNodeConnectorCache.getIfPresent(connector.getNodeConnectorRef()) != null){
            deletedNodeConnectorCache.invalidate(connector.getNodeConnectorRef());
        }
        LOG.debug("Node connector updated notification received.");
        manager.enqueue(new InventoryOperation() {
            @Override
            public void applyOperation(final ReadWriteTransaction tx) {
                final NodeConnectorRef ref = connector.getNodeConnectorRef();
                final NodeConnectorBuilder data = new NodeConnectorBuilder(connector);
                data.setKey(new NodeConnectorKey(connector.getId()));

                final FlowCapableNodeConnectorUpdated flowConnector = connector
                        .getAugmentation(FlowCapableNodeConnectorUpdated.class);
                if (flowConnector != null) {
                    final FlowCapableNodeConnector augment = InventoryMapping.toInventoryAugment(flowConnector);
                    data.addAugmentation(FlowCapableNodeConnector.class, augment);
                }
                InstanceIdentifier<NodeConnector> value = (InstanceIdentifier<NodeConnector>) ref.getValue();
                LOG.debug("updating node connector : {}.", value);
                NodeConnector build = data.build();
                tx.merge(LogicalDatastoreType.OPERATIONAL, value, build, true);
            }
        });
}

```

#### 4总结

对于前端需要显示的拓扑信息是由network-topology数据节点提供的，在network-topology数据节点下存在两类数据，一种是Link，一种是node，此时的node包括主机host和switch节点，host节点id是以host：开头，而switch节点的id是以openflow：开头，这个跟inventory数据节点的node是不一样的。

另外当switch设备（node）down掉之后，NodeChangeListenerImpl的processRemovedNode函数会将该node节点移除，同时与之相关的link（比如之前已在该node下发现host主机），并且会引起hosttracker当中的HostTrackerImpl删除该switch下的host主机，这样在拓扑上就不会有孤立的主机节点，但是该hosttracker还存在bug（已提交到odlbug系统），使用mininet测试，一旦模拟的host数量过多（200多个）时，down掉switch，会有1-2个host节点残余在拓扑上，通过代码和log查看，并不是调用删除处有问题，而是调用的odl基础框架有问题，还在进一步跟踪当中。



![ 我要小额赞助，鼓励作者写出更好的教程](https://raw.githubusercontent.com/xiaohongaiweno/blog/master/assets/img/%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98%E7%A0%81.png)


![ 我要小额赞助，鼓励作者写出更好的教程](https://raw.githubusercontent.com/xiaohongaiweno/blog/master/assets/img/%E6%94%AF%E4%BB%98%E5%AE%9D%E6%94%B6%E6%AC%BE%E7%A0%81.png)