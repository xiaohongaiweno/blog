---
layout: post
title: topology-manager�ܽ�
tags:
- ODL������
- SDN����
categories: SDN����
description: �������϶���sdn������ص����Ͻ������ҵ���״������ƪ���¿�ʼ���ҽ�½����������sdn���������еľ��顣
---

**topology-manager�ܽ�**

#### 0ǰ��

topology-managerģ��Ҳ����Ϊopenflowplugin��Ӧ�ò����APP����λ��openflowplugin-release-lithium-sr3�ļ��е�applications���У�������operational���ݿ��µ�network-topology:network-topology���ݽڵ㣨datastore���ݿ⣩����ɾ�Ĳ飬����odl���������ּ�һ̨����host���¼������뽻������link���ӵȡ���ʾ���˵�ǰ����Ҫ�Ӹ����ݽڵ��ϻ�ȡ�������߽������ڵ����ݲ��ܻ�����������ͼ����������ͼ��Դ�������棬һ������ͨ��LLDP���ֵ�switch�豸�Լ����link���ӣ���һ������ͨ��L2switch��hosttrackerģ�鷢�ֵĹ���switch�ϵ�host�����Լ�������ӡ�

#### 1 YANG����ģ��

network-topology:network-topology���ݽڵ���ֱ��ʹ�����ⲿietf-topology�����yang�ļ�����Ҫ���������ṹΪ��

container network-topology {

        list topology {

            description &quot;

                This is the model of an abstract topology.

                A topology contains nodes and links.

                Each topology MUST be identified by

                unique topology-id for reason that a network could contain many

                topologies.&quot;;

            key &quot;topology-id&quot;;

            leaf topology-id {

                type topology-id;

                description &quot;

                    It is presumed that a datastore will contain many topologies. To

                    distinguish between topologies it is vital to have UNIQUE

                    topology identifiers.&quot;;

            }

            list node {

                description &quot;The list of network nodes defined for the topology.&quot;;

                key &quot;node-id&quot;;

                uses node-attributes;

                must &quot;boolean(../underlay-topology[\*]/node[./supporting-nodes/node-ref])&quot;;

                    // This constraint is meant to ensure that a referenced node is in fact

                    // a node in an underlay topology.

                list termination-point {

                    description

                        &quot;A termination point can terminate a link.

                        Depending on the type of topology, a termination point could,

                        for example, refer to a port or an interface.&quot;;

                    key &quot;tp-id&quot;;

                    uses tp-attributes;

                }

            }

            list link {

                description &quot;

                    A Network Link connects a by Local (Source) node and

                    a Remote (Destination) Network Nodes via a set of the

                    nodes&#39; termination points.

                    As it is possible to have several links between the same

                    source and destination nodes, and as a link could potentially

                    be re-homed between termination points, to ensure that we

                    would always know to distinguish between links, every link

                    is identified by a dedicated link identifier.

                    Note that a link models a point-to-point link, not a multipoint

                    link.

                    Layering dependencies on links in underlay topologies are

                    not represented as the layering information of nodes and of

                    termination points is sufficient.&quot;;

                key &quot;link-id&quot;;

                uses link-attributes;

                must &quot;boolean(../underlay-topology/link[./supporting-link])&quot;;

                    // Constraint: any supporting link must be part of an underlay topology

                must &quot;boolean(../node[./source/source-node])&quot;;

                    // Constraint: A link must have as source a node of the same topology

                must &quot;boolean(../node[./destination/dest-node])&quot;;

                    // Constraint: A link must have as source a destination of the same topology

                must &quot;boolean(../node/termination-point[./source/source-tp])&quot;;

                    // Constraint: The source termination point must be contained in the source node

                must &quot;boolean(../node/termination-point[./destination/dest-tp])&quot;;

                    // Constraint: The destination termination point must be contained

                    // in the destination node

            }

        }

#### 2�������

��topology-manager�Ĵ���������Ҫ�����������ߣ���һ����applications�ļ����µ�topology-lldp-discoveryģ��ᴦ�������뽻������link����(���������lldp-speaker����lldp����̽���¼�����豸�Ƿ�Ϊopenflow������������ǽ���������topology-lldp-discovery��������link���ֵ�ͨ��)�����topology-lldp-discoveryģ��ᷢ������ͨ��FlowTopologyDiscoveryListener��

onLinkDiscovered    //�������

onLinkRemoved     //�����Ƴ�

�ڶ���topology-manager��Ҫ����Nodes/Node/NodeConnector/FlowCapableNodeConnector��Nodes/Node/NodeConnector/FlowCapableNode���ݵı仯�������ݾ�����inventory-managerģ����ӵĽ�������Ϣ�ڵ㣬���Ե�inventory-managerģ���¼ӻ����Ƴ�һ̨�������ڵ�ʱ���ᴥ��Nodes/Node/NodeConnector/FlowCapableNodeConnector

�Լ�Nodes/Node/NodeConnector/FlowCapableNode���ݽڵ�ı仯����������topology-managerģ�����

processAddedTerminationPoints

processUpdatedTerminationPoints

processRemovedTerminationPoints

processAddedNode

processRemovedNode

�������̡�

#### 3��������

topology-manager�������TopologyLldpDiscoveryImplModule.java�ļ���createInstance()����������ú�����new��FlowCapableTopologyProvider����Ȼ��ע��FlowTopologyDiscoveryListenerͨ��Ľ�����FlowCapableTopologyExporter��ͬʱ���ü���Nodes/Node/NodeConnector/FlowCapableNodeConnector

�Լ�Nodes/Node/NodeConnector/FlowCapableNode�����ݱ仯��

�ȷ���FlowTopologyDiscoveryListenerͨ��Ľ�����FlowCapableTopologyExporter��

###### 3.1 onLinkDiscovered�¼�

��������ͨ��LLDP�����¼ӵ��豸�������豸����һ����·ʱ���������뽻����֮�����·�����ᴥ��onLinkDiscovered�����ĵ��ã�����link���������ݿ⵱�С�

public void onLinkDiscovered(final LinkDiscovered notification) {

        processor.enqueueOperation(new TopologyOperation() {

            @Override

            public void applyOperation(final ReadWriteTransaction transaction) {

                //����network-topology��link����

                final Link link = toTopologyLink(notification);

                //����Ӧ�ô�ŵ��������ڵ�λ��

                final InstanceIdentifier&lt;Link&gt; path = TopologyManagerUtil.linkPath(link, iiToTopology);

                transaction.merge(LogicalDatastoreType.OPERATIONAL, path, link, true);

            }

            @Override

            public String toString() {

                return &quot;onLinkDiscovered&quot;;

            }

        });

    }

###### 3.2 onLinkRemoved�¼�

���������ӶϿ�֮�󣬾ͻᴥ��onLinkRemoved�����ĵ��ã���֮ǰ���������ݿ⵱�е�link��Ϣɾ����

public void onLinkRemoved(final LinkRemoved notification) {

        processor.enqueueOperation(new TopologyOperation() {

            @Override

            public void applyOperation(final ReadWriteTransaction transaction) {

                Optional&lt;Link&gt; linkOptional = Optional.absent();

                try {

                    // read that checks if link exists (if we do not do this we might get an exception on delete)

                    linkOptional = transaction.read(LogicalDatastoreType.OPERATIONAL,

                            TopologyManagerUtil.linkPath(toTopologyLink(notification), iiToTopology)).checkedGet();

                } catch (ReadFailedException e) {

                    LOG.warn(&quot;Error occured when trying to read Link: {}&quot;, e.getMessage());

                    LOG.debug(&quot;Error occured when trying to read Link.. &quot;, e);

                }

                //ɾ��network-topology�µ�link����

                if (linkOptional.isPresent()) {

                    transaction.delete(LogicalDatastoreType.OPERATIONAL, TopologyManagerUtil.linkPath(toTopologyLink(notification), iiToTopology));

                }

            }

            @Override

            public String toString() {

                return &quot;onLinkRemoved&quot;;

            }

        });

    }

###### 3.3 FlowCapableNode���ݱ仯�¼�

������������Nodes/Node/NodeConnector/FlowCapableNode�����ݱ仯��NodeChangeListenerImpl����Nodes/Node/NodeConnector/FlowCapableNode�����ݷ����仯��inventory-manager�����¼�/ɾ��һ̨�������ڵ㣩���ᴥ��NodeChangeListenerImpl��onDataChanged�������ã��ڸú������е�����������������

processAddedNode(change.getCreatedData());

processRemovedNode(change.getRemovedPaths());

�����������Լ��ڲ��ж��Ƿ��ǽڵ����/ɾ���¼������ս��������ڵ���Ϣд��network-topology���������С�

protected void createData(final InstanceIdentifier&lt;?&gt; iiToNodeInInventory) {

        final NodeId nodeIdInTopology = provideTopologyNodeId(iiToNodeInInventory);

        if (nodeIdInTopology != null) {

            final InstanceIdentifier&lt;org.opendaylight.yang.gen.v1.urn.tbd.params.xml.ns.yang.network.topology.rev131021.network.topology.topology.Node&gt; iiToTopologyNode = provideIIToTopologyNode(nodeIdInTopology);

            //д��network-topology������

            sendToTransactionChain(prepareTopologyNode(nodeIdInTopology, iiToNodeInInventory), iiToTopologyNode);

        } else {

            LOG.debug(&quot;Inventory node key is null. Data can&#39;t be written to topology&quot;);

        }

    }

###### 3.4 FlowCapableNodeConnector���ݱ仯�¼�

������Nodes/Node/NodeConnector/FlowCapableNodeConnector�����ݼ�����TerminationPointChangeListenerImpl����Nodes/Node/NodeConnector/FlowCapableNodeConnector���ݷ����仯ʱ���ᴥ��onDataChanged��������

processAddedTerminationPoints(change.getCreatedData());

processUpdatedTerminationPoints(change.getUpdatedData());

processRemovedTerminationPoints(change.getRemovedPaths());

���Ƶģ�����3������Ҳ���ڲ��ж��Ƿ��������¼�/����/ɾ���¼����������¼��¼��������createData������TerminationPointд��network-topology��������

protected void createData(final InstanceIdentifier&lt;?&gt; iiToNodeInInventory, final DataObject data) {

        TpId terminationPointIdInTopology = provideTopologyTerminationPointId(iiToNodeInInventory);

        if (terminationPointIdInTopology != null) {

            InstanceIdentifier&lt;TerminationPoint&gt; iiToTopologyTerminationPoint = provideIIToTopologyTerminationPoint(

                    terminationPointIdInTopology, iiToNodeInInventory);

            TerminationPoint point = prepareTopologyTerminationPoint(terminationPointIdInTopology, iiToNodeInInventory);

            sendToTransactionChain(point, iiToTopologyTerminationPoint);

            if (data instanceof FlowCapableNodeConnector) {

                removeLinks((FlowCapableNodeConnector) data, point);

            }

        } else {

            LOG.debug(&quot;Inventory node connector key is null. Data can&#39;t be written to topology termination point&quot;);

        }

}

#### 4�ܽ�

����ǰ����Ҫ��ʾ��������Ϣ����network-topology���ݽڵ��ṩ�ģ���network-topology���ݽڵ��´����������ݣ�һ����Link��һ����node����ʱ��node��������host��switch�ڵ㣬host�ڵ�id����host����ͷ����switch�ڵ��id����openflow����ͷ�������inventory���ݽڵ��node�ǲ�һ���ġ�

���⵱switch�豸��node��down��֮��NodeChangeListenerImpl��processRemovedNode�����Ὣ��node�ڵ��Ƴ���ͬʱ��֮��ص�link������֮ǰ���ڸ�node�·���host�����������һ�����hosttracker���е�HostTrackerImplɾ����switch�µ�host�����������������ϾͲ����й����������ڵ㣬���Ǹ�hosttracker������bug�����ύ��odlbugϵͳ����ʹ��mininet���ԣ�һ��ģ���host�������ࣨ200�����ʱ��down��switch������1-2��host�ڵ�����������ϣ�ͨ�������log�鿴�������ǵ���ɾ���������⣬���ǵ��õ�odl������������⣬���ڽ�һ�����ٵ��С�