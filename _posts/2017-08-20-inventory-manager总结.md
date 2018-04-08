---
layout: post
title: inventory-manager总结
tags:
- ODL控制器
- SDN开发
categories: SDN开发
description: 鉴于网上对于sdn开发相关的资料较少又乱的现状，从这篇文章开始，我将陆续分享我在sdn开发过程中的经验。
---

**inventory-manager总结**

#### 0前言

inventory-manager模块作为openflowplugin的应用层程序（可理解成odl的APP），位于openflowplugin-release-lithium-sr3文件夹的applications当中，负责处理operational数据库下的opendaylight-inventory数据节点（datastore数据库）的增删改查，比如odl控制器新加一台openflow交换机或者移除一台交换机。同时opendaylight-inventory数据节点还为前端提供数据支撑。

#### 1 YANG数据模型

ODL在controller\opendaylight\md-sal\model\model-inventory\src\main\yang\opendaylight-inventory.yang文件当中定义了inventory的数据节点的树形结构：

    container nodes {

        description &quot;The root container of all nodes.&quot;;

        list node {

            key &quot;id&quot;;

            ext:context-instance &quot;node-context&quot;;

            description &quot;A list of nodes (as defined by the &#39;grouping node&#39;).&quot;;

            uses node; //this refers to the &#39;grouping node&#39; defined above.

        }

    }

    grouping node {

        description &quot;Describes the contents of a generic node -

                     essentially an ID and a list of node-connectors.

                     Acts as an augmentation point where other yang files

                      can add additional information.&quot;;

        leaf id {

            type node-id;

            description &quot;The unique identifier for the node.&quot;;

        }

        list &quot;node-connector&quot; {

            key &quot;id&quot;;

            description &quot;A list of node connectors that belong this node.&quot;;

            ext:context-instance &quot;node-connector-context&quot;;

            uses node-connector;

        }

}

    grouping node-connector {

        description &quot;Describes a generic node connector which consists of an ID.

                     Acts as an augmentation point where other yang files can

                      add additional information.&quot;;

        leaf id {

            type node-connector-id;

            description &quot;The unique identifier for the node-connector.&quot;;

        }

}

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

        LOG.debug(&quot;Node updated notification received,{}&quot;, node.getNodeRef().getValue());

        manager.enqueue(new InventoryOperation() {

            @Override

            public void applyOperation(ReadWriteTransaction tx) {

                final NodeRef ref = node.getNodeRef();

                @SuppressWarnings(&quot;unchecked&quot;)

                //组装node数据

                InstanceIdentifierBuilder&lt;Node&gt; builder = ((InstanceIdentifier&lt;Node&gt;) ref.getValue()).builder();

                InstanceIdentifierBuilder&lt;FlowCapableNode&gt; augmentation = builder.augmentation(FlowCapableNode.class);

                final InstanceIdentifier&lt;FlowCapableNode&gt; path = augmentation.build();

                CheckedFuture&lt;Optional&lt;FlowCapableNode&gt;, ?&gt; readFuture = tx.read(LogicalDatastoreType.OPERATIONAL, path);

                Futures.addCallback(readFuture, new FutureCallback&lt;Optional&lt;FlowCapableNode&gt;&gt;() {

                    @Override

                    public void onSuccess(Optional&lt;FlowCapableNode&gt; optional) {

                        enqueueWriteNodeDataTx(node, flowNode, path);

                        if (!optional.isPresent()) {

                            enqueuePutTable0Tx(ref);

                        }

                    }

                    @Override

                    public void onFailure(Throwable throwable) {

                        LOG.debug(String.format(&quot;Can&#39;t retrieve node data for node %s. Writing node data with table0.&quot;, node));

                        enqueueWriteNodeDataTx(node, flowNode, path);

                        enqueuePutTable0Tx(ref);

                    }

                });

            }

        });

    }

###### 3.2 onNodeRemoved事件

ODL控制器与openflow交换机断开连接之后，就会触发onNodeRemoved函数，此时Inventory-manager模块需要将该节点从数据库datastore删除该node节点。

    public synchronized void onNodeRemoved(final NodeRemoved node) {

        if(deletedNodeCache.getIfPresent(node.getNodeRef()) == null){

            deletedNodeCache.put(node.getNodeRef(), Boolean.TRUE);

        } else {

            //its been noted that creating an operation for already removed node, fails

            // the entire transaction chain, there by failing deserving removals

            LOG.debug(&quot;Already received notification to remove node, {} - Ignored&quot;,

                    node.getNodeRef().getValue());

            return;

        }

        LOG.debug(&quot;Node removed notification received, {}&quot;, node.getNodeRef().getValue());

        manager.enqueue(new InventoryOperation() {

            @Override

            public void applyOperation(final ReadWriteTransaction tx) {

                final NodeRef ref = node.getNodeRef();

                LOG.debug(&quot;removing node : {}&quot;, ref.getValue());

                tx.delete(LogicalDatastoreType.OPERATIONAL, ref.getValue());

            }

        });

    }

###### 3.3 onNodeConnectorRemoved事件

当openflow交换机端口变成down之后，就会触发onNodeConnectorRemoved函数，此时Inventory-manager模块需要将该节点从数据库datastore删除该NodeConnector节点。

public synchronized void onNodeConnectorRemoved(final NodeConnectorRemoved connector) {

        if(deletedNodeConnectorCache.getIfPresent(connector.getNodeConnectorRef()) == null){

            deletedNodeConnectorCache.put(connector.getNodeConnectorRef(), Boolean.TRUE);

        } else {

            //its been noted that creating an operation for already removed node-connectors, fails

            // the entire transaction chain, there by failing deserving removals

            LOG.debug(&quot;Already received notification to remove nodeConnector, {} - Ignored&quot;,

                    connector.getNodeConnectorRef().getValue());

            return;

        }

        LOG.debug(&quot;Node connector removed notification received, {}&quot;, connector.getNodeConnectorRef().getValue());

        manager.enqueue(new InventoryOperation() {

            @Override

            public void applyOperation(final ReadWriteTransaction tx) {

                final NodeConnectorRef ref = connector.getNodeConnectorRef();

                LOG.debug(&quot;removing node connector {} &quot;, ref.getValue());

                tx.delete(LogicalDatastoreType.OPERATIONAL, ref.getValue());

            }

        });

}

###### 3.4 onNodeConnectorUpdated事件

当openflow交换机端口变成up之后，就会触发onNodeConnectorUpdated函数，此时Inventory-manager模块需要添加该NodeConnector节点到数据库datastore。

public synchronized void onNodeConnectorUpdated(final NodeConnectorUpdated connector) {

        if (deletedNodeConnectorCache.getIfPresent(connector.getNodeConnectorRef()) != null){

            deletedNodeConnectorCache.invalidate(connector.getNodeConnectorRef());

        }

        LOG.debug(&quot;Node connector updated notification received.&quot;);

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

                InstanceIdentifier&lt;NodeConnector&gt; value = (InstanceIdentifier&lt;NodeConnector&gt;) ref.getValue();

                LOG.debug(&quot;updating node connector : {}.&quot;, value);

                NodeConnector build = data.build();

                tx.merge(LogicalDatastoreType.OPERATIONAL, value, build, true);

            }

        });

}

#### 4总结

 inventory-manager作为ODL的APP形式管理inventory节点信息，位于Openflowplguin的applications文件夹，而Openflowplguin模块当中的applications是建立在Openflowplguin基础模块之上的，比如openflowplugin-impl、openflowplugin等，由Openflowplguin基础模块处理好openflow协议最后发出node新加、更新、删除等通告。