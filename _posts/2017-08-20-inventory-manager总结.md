---
layout: post
title: inventory-manager总结
tags:
- ODL控制器
- SDN开发
categories: SDN开发
description: 鉴于网上对于sdn开发相关的资料较少又乱的现状，从这篇文章开始，我将陆续分享我在sdn开发过程中的经验。
---

inventory-manager模块作为openflowplugin的应用层程序（可理解成odl的APP），位于openflowplugin-release-lithium-sr3文件夹的applications当中，负责处理operational数据库下的opendaylight-inventory数据节点（datastore数据库）的增删改查，比如odl控制器新加一台openflow交换机或者移除一台交换机。同时opendaylight-inventory数据节点还为前端提供数据支撑。

#### 1 YANG数据模型

ODL在controller\opendaylight\md-sal\model\model-inventory\src\main\yang\opendaylight-inventory.yang文件当中定义了inventory的数据节点的树形结构：
<pre>
<code>
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
</code>
</pre>

并且在yang语法当中还可以使用Augmentation来在某个节点下插入其他数据。同时还定义了以下notification通告：

node-updated    //交换机节点添加

node-removed    //交换机节点删除

node-connector-updated     //交换机端口up

node-connector-removed     //交换机端口down

这样在openflowplugin基础模块中发通告，在inventory-manager中接收通告并处理交换机节点信息。

#### 2相关依赖

ODL控制器与openflow交换机通过openflow协议建立连接（此链接已经实现TLS加密，可以在配置文件当中启用）之后，openflowplugin会通过notification的形式发布通告，具体在org.opendaylight.openflowplugin.openflow.md.core.sal包下的SalRegistrationManager.java文件当中的以下两个函数。

public void onSessionAdded(final SwitchSessionKeyOF sessionKey, final SessionContext context)

public void onSessionRemoved(final SessionContext context)

例如进入onSessionAdded函数，将会发出NodeUpdated的notification，而inventory-manager管理模块因为事先注册接收该通告，所以能够收到该通告并进入inventory-manager的处理逻辑。

#### 3代码流程

inventory-manager的入口在InventoryManagerImplModule.java文件的createInstance（）函数，进入该函数后new出NodeChangeCommiter对象，并注册OpendaylightInventoryListener通告的接收器，因此NodeChangeCommiter对象专门处理以下几类通告：

onNodeUpdated     //交换机节点上线事件

onNodeRemoved     //交换机节点下线事件

onNodeConnectorRemoved   //交换机端口down事件

onNodeConnectorUpdated    //交换机端口up事件

###### 3.1 onNodeUpdated事件

当ODL控制器与openflow交换机建立连接之后，就会触发onNodeUpdated函数，此时Inventory-manager模块做的事情就是将该节点写入yang的node节点（datastore数据库）。
<pre>
<code>
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

</code>
</pre>

###### 3.2 onNodeRemoved事件

ODL控制器与openflow交换机断开连接之后，就会触发onNodeRemoved函数，此时Inventory-manager模块需要将该节点从数据库datastore删除该node节点。
<pre>
<code>
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

</code>
</pre>


###### 3.3 onNodeConnectorRemoved事件

当openflow交换机端口变成down之后，就会触发onNodeConnectorRemoved函数，此时Inventory-manager模块需要将该节点从数据库datastore删除该NodeConnector节点。
<pre>
<code>
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

</code>
</pre>

###### 3.4 onNodeConnectorUpdated事件

当openflow交换机端口变成up之后，就会触发onNodeConnectorUpdated函数，此时Inventory-manager模块需要添加该NodeConnector节点到数据库datastore。
<pre>
<code>
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

</code>
</pre>

#### 4总结

 inventory-manager作为ODL的APP形式管理inventory节点信息，位于Openflowplguin的applications文件夹，而Openflowplguin模块当中的applications是建立在Openflowplguin基础模块之上的，比如openflowplugin-impl、openflowplugin等，由Openflowplguin基础模块处理好openflow协议最后发出node新加、更新、删除等通告。
 


![ 我要小额赞助，鼓励作者写出更好的教程](https://raw.githubusercontent.com/xiaohongaiweno/blog/master/assets/img/%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98%E7%A0%81.png)


![ 我要小额赞助，鼓励作者写出更好的教程](https://raw.githubusercontent.com/xiaohongaiweno/blog/master/assets/img/%E6%94%AF%E4%BB%98%E5%AE%9D%E6%94%B6%E6%AC%BE%E7%A0%81.png)