<script>
import { VueFlow, Handle } from '@vue-flow/core';
import { Background } from '@vue-flow/background';
import { Controls } from '@vue-flow/controls';
import { MiniMap } from '@vue-flow/minimap';
import { allHash } from '@shell/utils/promise';
import Checkbox from '@components/Form/Checkbox/Checkbox.vue';
import { HCI } from '../types';

// --- Constants & Configuration ---
const PROVIDERS = {
  OVN:     'ovn',
  OVERLAY: 'overlay',
};

const COLORS = {
  VPC:       '#2453ff',
  SUBNET:    '#fe7c3f',
  OVERLAY:   '#9333ea',
  VM:        '#00bda7',
  STOPPED:   '#9ca3af',
  TEXT:      '#000000',
  MUTED:     '#6b7280',
  BG_STOP:   '#f3f4f6',
  LINK_GRAY: '#9ca3af', // Neutral color for structural links
};

const LAYOUT_CONFIG = {
  VMS_PER_ROW:    5,
  BASE_PADDING:   24,
  VERTICAL_GAP:   40,
  HORIZONTAL_GAP: 30,
  VPC_WIDTH:      220,
  MIN_NODE_WIDTH: 140,
  VM_WIDTH:       200,
};

const DEFAULT_VALUES = {
  CIDR:            'N/A',
  CLUSTER_NETWORK: 'mgmt',
  INTERFACE:       'eth0',
  MAC:             'N/A',
};

export default {
  name: 'VPCDetail',

  components: {
    VueFlow,
    Handle,
    Background,
    Controls,
    MiniMap,
    Checkbox,
  },

  props: {
    value: {
      type:     Object,
      required: true,
    },
  },

  data() {
    return {
      nodes:   [],
      edges:   [],
      loading: true,

      // Interaction State
      selectedNodeId: null,
      relatedIds:     new Set(),

      // Visibility Controls Models
      showVPC:      true,
      showSubnets:  true,
      showVMs:      true,
      showOverlays: true,
    };
  },

  created() {
    // Performance: Canvas context doesn't need to be reactive
    this.canvasContext = null;
  },

  async fetch() {
    const inStore = this.$store.getters['currentProduct'].inStore;

    try {
      await allHash({
        subnets:            this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.SUBNET }),
        ips:                this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.IP }),
        vms:                this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.VM }),
        clusterNetworks:    this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.CLUSTER_NETWORK }),
        networkAttachments: this.$store
          .dispatch(`${ inStore }/findAll`, { type: HCI.NETWORK_ATTACHMENT })
          .catch((e) => {
            // eslint-disable-next-line no-console
            console.warn('Network attachments not available:', e.message);

            return [];
          }),
      });
    } catch (error) {
      // eslint-disable-next-line no-console
      console.error('Failed to fetch VPC resources:', error);
      throw error;
    }

    await this.loadTopology();
  },

  computed: {
    // --- Store Getters ---
    inStore() {
      return this.$store.getters['currentProduct']?.inStore || 'cluster';
    },
    allSubnets() {
      return this.$store.getters[`${ this.inStore }/all`](HCI.SUBNET) || [];
    },
    allIps() {
      return this.$store.getters[`${ this.inStore }/all`](HCI.IP) || [];
    },
    allVMs() {
      return this.$store.getters[`${ this.inStore }/all`](HCI.VM) || [];
    },
    allNetworkAttachments() {
      try {
        return (
          this.$store.getters[`${ this.inStore }/all`](HCI.NETWORK_ATTACHMENT) ||
          []
        );
      } catch (e) {
        return [];
      }
    },

    // --- Stats for Badges ---
    subnetCount() {
      return this.nodes.filter((n) => n.data?.type === 'subnet').length;
    },
    overlayCount() {
      return this.nodes.filter((n) => n.data?.type === 'overlay-network')
        .length;
    },
    vmCount() {
      return this.nodes.filter(
        (n) => n.data?.type === 'vm' || n.data?.type === 'multi-network-vm',
      ).length;
    },

    // --- Styles ---
    backgroundPatternColor() {
      return this.getCssVar('--border') || '#f1f1f1';
    },
    layoutConfig() {
      return LAYOUT_CONFIG;
    },
    colors() {
      return COLORS;
    },

    // --- Visibility Options Configuration ---
    visibilityOptions() {
      return [
        {
          modelKey:      'showVPC',
          label:         'VPC',
          badgeColorKey: 'VPC',
          count:         1,
        },
        {
          modelKey:      'showSubnets',
          label:         'Subnets',
          badgeColorKey: 'SUBNET',
          count:         this.subnetCount,
        },
        {
          modelKey:      'showOverlays',
          label:         'Overlay Networks',
          badgeColorKey: 'OVERLAY',
          count:         this.overlayCount,
          // UX: 當 Subnets 被隱藏時，Overlay 選項被禁用
          disabled:      !this.showSubnets,
        },
        {
          modelKey:      'showVMs',
          label:         'VMs',
          badgeColorKey: 'VM',
          count:         this.vmCount,
        },
      ];
    },

    // --- Node & Edge Filtering Logic ---
    filteredNodes() {
      return this.nodes
        .filter((node) => {
          const type = node.data?.type;

          if (type === 'vpc' && !this.showVPC) return false;

          // Logic: Hide Subnet, Group and Overlay if showSubnets is false
          if (
            (type === 'subnet' ||
              type === 'group' ||
              type === 'overlay-network') &&
            !this.showSubnets
          ) {
            return false;
          }

          if ((type === 'vm' || type === 'multi-network-vm') && !this.showVMs) {
            return false;
          }
          if (type === 'overlay-network' && !this.showOverlays) return false;

          return true;
        })
        .map((node) => {
          // Calculate Highlight State
          const isTarget = this.selectedNodeId === node.id;
          const isRelated =
            this.selectedNodeId &&
            (this.relatedIds.has(node.id) || node.data?.type === 'vpc');
          const isGroup = node.data?.type === 'group';

          // Determine "Dimmed" state: Highlight is active, but this node is NOT target/related/group
          const isDimmed =
            this.selectedNodeId && !isGroup && !isRelated && !isTarget;

          let stateClass = '';
          let zIndex = node.zIndex || 1;

          if (this.selectedNodeId) {
            if (isTarget) {
              stateClass = 'node-focused';
              zIndex = 1000;
            } else if (isRelated) {
              stateClass = 'node-related';
              zIndex = 999;
            } else if (isDimmed) {
              stateClass = 'node-dimmed';
              zIndex = 0;
            }
          }

          return {
            ...node,
            data:  { ...node.data, stateClass },
            style: { ...node.style, zIndex },
          };
        });
    },

    filteredEdges() {
      const visibleNodeIds = new Set(this.filteredNodes.map((n) => n.id));

      return this.edges
        .filter(
          (edge) => visibleNodeIds.has(edge.source) && visibleNodeIds.has(edge.target),
        )
        .map((edge) => {
          const isRelated =
            !this.selectedNodeId ||
            (this.relatedIds.has(edge.source) &&
              this.relatedIds.has(edge.target));

          return {
            ...edge,
            class:    isRelated ? '' : 'dimmed',
            animated: edge.animated && isRelated,
          };
        });
    },
  },

  watch: {
    allSubnets: {
      handler() {
        if (!this.loading) this.loadTopology();
      },
      deep: true,
    },
    allIps: {
      handler() {
        if (!this.loading) this.loadTopology();
      },
      deep: true,
    },
    allVMs: {
      handler() {
        if (!this.loading) this.loadTopology();
      },
      deep: true,
    },
    'value.metadata.resourceVersion'() {
      if (!this.loading) this.loadTopology();
    },

    // --- UX Update: Sync Overlay state with Subnet state ---
    showSubnets(val) {
      // Checked Subnet -> Checked Overlay
      // Unchecked Subnet -> Unchecked Overlay
      this.showOverlays = val;
    },
  },

  methods: {
    // ==========================================
    // 1. Data Processing & Preparation
    // ==========================================

    prepareResourceMaps(vpcName, subnets, ips, vms) {
      const vpcSubnets = (Array.isArray(subnets) ? subnets : []).filter(
        (subnet) => subnet.spec?.vpc === vpcName,
      );

      const vmToSubnets = {};
      const vmToIps = {};
      const vmToInterfaceDetails = {};

      (Array.isArray(ips) ? ips : []).forEach((ip) => {
        const podName = ip.spec?.podName;
        const subnet = ip.spec?.subnet;

        if (podName && subnet) {
          const matchedVm = vms.find((vm) => vm.metadata?.name === podName);

          if (matchedVm) {
            if (!vmToSubnets[podName]) {
              vmToSubnets[podName] = [];
              vmToIps[podName] = [];
              vmToInterfaceDetails[podName] = [];
            }
            if (!vmToSubnets[podName].includes(subnet)) {
              vmToSubnets[podName].push(subnet);
              vmToIps[podName].push(ip.spec?.ipAddress);
              vmToInterfaceDetails[podName].push({
                mac: ip.spec?.macAddress || DEFAULT_VALUES.MAC,
                ip:  ip.spec?.ipAddress,
                subnet,
              });
            }
          }
        }
      });

      const vpcVMs = (Array.isArray(vms) ? vms : []).filter((vm) => {
        const vmName = vm.metadata?.name;
        const vmSubnets = vmToSubnets[vmName] || [];

        return (
          vmSubnets.length > 0 &&
          vmSubnets.some((vmSubnet) => vpcSubnets.some((s) => s.metadata.name === vmSubnet),
          )
        );
      });

      return {
        vpcSubnets,
        vmToSubnets,
        vmToIps,
        vmToInterfaceDetails,
        vpcVMs,
      };
    },

    extractNADInfo(provider) {
      const nad = this.allNetworkAttachments.find(
        (n) => n.id === provider || n.metadata?.name === provider,
      );

      let nadType = PROVIDERS.OVERLAY;
      let nadClusterNetwork = DEFAULT_VALUES.CLUSTER_NETWORK;
      let nadName = provider.split('/').pop();

      if (nad) {
        try {
          if (nad.spec?.config) {
            const config = JSON.parse(nad.spec.config);

            nadType = config.type || PROVIDERS.OVERLAY;
          }
        } catch (e) {
          // eslint-disable-next-line no-console
          console.warn('Failed to parse NAD config', e);
        }
        nadClusterNetwork =
          nad.metadata?.labels?.['network.harvesterhci.io/clusternetwork'] ||
          DEFAULT_VALUES.CLUSTER_NETWORK;
        nadName = nad.metadata?.name || nadName;
      }

      return {
        nadName,
        nadType,
        nadClusterNetwork,
      };
    },

    // ==========================================
    // 2. Dimension Calculation (Canvas API)
    // ==========================================

    getTextWidth(text, font) {
      if (!this.canvasContext) {
        const canvas = document.createElement('canvas');

        this.canvasContext = canvas.getContext('2d');
      }
      this.canvasContext.font = font;
      const metrics = this.canvasContext.measureText(text);

      return metrics.width;
    },

    calculateNodeDimensions(lines, fontSize, fontWeight = 'normal') {
      const padding = LAYOUT_CONFIG.BASE_PADDING;
      const minWidth = LAYOUT_CONFIG.MIN_NODE_WIDTH;
      const fontString = `${ fontWeight } ${ fontSize }px sans-serif`;

      const maxTextWidth = Math.max(
        ...lines.map((line) => this.getTextWidth(line, fontString)),
      );

      const width = Math.max(maxTextWidth + padding * 2, minWidth);
      const lineHeight = fontSize * 1.4;
      const height = lines.length * lineHeight + padding * 2 + 4; // +4 for borders

      return { width, height };
    },

    calculateSubnetDimensions(name, cidr, provider) {
      // Title: Bold 18px, Details: Normal 14px (Simulated by taking max width)
      const titleWidth = this.getTextWidth(name, '700 18px sans-serif');
      const detailsWidth = Math.max(
        this.getTextWidth(`CIDR: ${ cidr }`, '400 14px sans-serif'),
        this.getTextWidth(`Provider: ${ provider }`, '400 14px sans-serif'),
      );

      const contentWidth =
        Math.max(titleWidth, detailsWidth) + LAYOUT_CONFIG.BASE_PADDING * 2;
      const width = Math.max(contentWidth, LAYOUT_CONFIG.MIN_NODE_WIDTH);

      // Fixed height estimation: Title + 2 lines
      const height =
        18 * 1.4 + 14 * 1.4 * 2 + LAYOUT_CONFIG.BASE_PADDING * 2 + 10;

      return { width, height };
    },

    calculateOverlayDimensions(name, type, network) {
      const titleWidth = this.getTextWidth(name, '700 18px sans-serif');
      const detailsWidth = Math.max(
        this.getTextWidth(`Type: ${ type }`, '400 14px sans-serif'),
        this.getTextWidth(`Cluster Network: ${ network }`, '400 14px sans-serif'),
      );

      const contentWidth =
        Math.max(titleWidth, detailsWidth) + LAYOUT_CONFIG.BASE_PADDING * 2;
      const width = Math.max(contentWidth, LAYOUT_CONFIG.MIN_NODE_WIDTH);

      const height =
        18 * 1.4 + 14 * 1.4 * 2 + LAYOUT_CONFIG.BASE_PADDING * 2 + 10;

      return { width, height };
    },

    calculateVMDimensions(name, status, interfaces) {
      const titleStr = name + status;
      const titleWidth = this.getTextWidth(titleStr, '700 18px sans-serif');

      let maxDetailWidth = 0;

      interfaces.forEach((iface) => {
        maxDetailWidth = Math.max(
          maxDetailWidth,
          this.getTextWidth(`IP: ${ iface.ip }`, '400 14px sans-serif'),
        );
        maxDetailWidth = Math.max(
          maxDetailWidth,
          this.getTextWidth(`MAC: ${ iface.mac }`, '400 14px sans-serif'),
        );
      });

      const contentWidth =
        Math.max(titleWidth, maxDetailWidth) + LAYOUT_CONFIG.BASE_PADDING * 2;
      const width = Math.max(contentWidth, LAYOUT_CONFIG.VM_WIDTH);

      const linesCount = 1 + interfaces.length * 2;
      const height =
        linesCount * 14 * 1.4 + LAYOUT_CONFIG.BASE_PADDING * 2 + 10;

      return { width, height };
    },

    // ==========================================
    // 3. Topology Building (Main Logic)
    // ==========================================

    async loadTopology() {
      if (this._loadingTopology) return;
      this._loadingTopology = true;
      this.loading = true;

      try {
        const vpc = this.value;

        // 1. Prepare Data Maps
        const {
          vpcSubnets,
          vmToSubnets,
          vmToIps,
          vmToInterfaceDetails,
          vpcVMs,
        } = this.prepareResourceMaps(
          vpc.metadata.name,
          this.allSubnets,
          this.allIps,
          this.allVMs,
        );

        const nodes = [];
        const edges = [];
        const subnetDimensionsCache = {};
        const subnetWidths = [];

        // 2. Pre-calculate Dimensions for Layout
        vpcSubnets.forEach((s) => {
          const sName = s.metadata.name;
          const provider = s.spec?.provider || PROVIDERS.OVN;
          const hasOverlay =
            provider && provider !== PROVIDERS.OVN && provider.trim() !== '';
          const providerDisplay = hasOverlay ? provider.split('/').pop() : PROVIDERS.OVN;
          const cidr = s.spec?.cidrBlock || DEFAULT_VALUES.CIDR;

          if (hasOverlay) {
            const { nadName, nadType, nadClusterNetwork } =
              this.extractNADInfo(provider);
            const subDims = this.calculateSubnetDimensions(
              sName,
              cidr,
              providerDisplay,
            );
            const overlayDims = this.calculateOverlayDimensions(
              nadName,
              nadType,
              nadClusterNetwork,
            );

            // Group dimensions = max child width + padding
            const gW =
              Math.max(subDims.width, overlayDims.width) +
              LAYOUT_CONFIG.BASE_PADDING * 2;
            const gH =
              subDims.height +
              LAYOUT_CONFIG.VERTICAL_GAP +
              overlayDims.height +
              LAYOUT_CONFIG.BASE_PADDING * 2;

            subnetDimensionsCache[sName] = {
              nadName,
              nadType,
              nadClusterNetwork,
              subnetWidth:   subDims.width,
              subnetHeight:  subDims.height,
              overlayWidth:  overlayDims.width,
              overlayHeight: overlayDims.height,
              groupWidth:    gW,
              groupHeight:   gH,
              hasOverlay:    true,
            };
            subnetWidths.push(gW);
          } else {
            const subDims = this.calculateSubnetDimensions(
              sName,
              cidr,
              providerDisplay,
            );

            subnetDimensionsCache[sName] = {
              subnetWidth: subDims.width,
              hasOverlay:  false,
            };
            subnetWidths.push(subDims.width);
          }
        });

        // 3. Calculate Global Layout Positions
        const layout = this.calculateGlobalLayout(
          subnetWidths,
          subnetDimensionsCache,
        );

        // 4. Create VPC Root Node
        const totalWidth =
          layout.subnetXPositions.length > 0 ? layout.subnetXPositions[layout.subnetXPositions.length - 1] +
              subnetWidths[subnetWidths.length - 1] : 200;

        nodes.push({
          id:       'vpc',
          type:     'vpc',
          position: { x: totalWidth / 2, y: layout.vpcY },
          data:     { type: 'vpc', name: vpc.metadata.name },
          style:    { width: `${ LAYOUT_CONFIG.VPC_WIDTH }px` },
        });

        // 5. Create Subnet Nodes (and Overlay Groups)
        vpcSubnets.forEach((subnet, index) => {
          const sName = subnet.metadata.name;
          const dims = subnetDimensionsCache[sName];
          const startX = layout.subnetXPositions[index];
          const providerDisplay = dims.hasOverlay ? subnet.spec?.provider?.split('/').pop() : PROVIDERS.OVN;

          this.buildSubnetNodes(
            nodes,
            edges,
            subnet,
            dims,
            startX,
            layout,
            providerDisplay,
          );
        });

        // 6. Create VM Nodes
        this.buildVMNodes(
          nodes,
          edges,
          vpcVMs,
          vmToSubnets,
          vmToIps,
          vmToInterfaceDetails,
          layout,
          vpcSubnets,
        );

        this.nodes = nodes;
        this.edges = edges;

        // 7. Post-render Adjustment (Center Alignment)
        this.$nextTick(() => {
          this.resizeGroups();
        });
      } catch (error) {
        // eslint-disable-next-line no-console
        console.error('Failed to load VPC topology:', error);
      } finally {
        this.loading = false;
        this._loadingTopology = false;
      }
    },

    calculateGlobalLayout(subnetWidths, subnetDimensionsCache) {
      const layout = {
        vpcY:    LAYOUT_CONFIG.BASE_PADDING,
        subnetY:
          LAYOUT_CONFIG.BASE_PADDING + 80 + LAYOUT_CONFIG.VERTICAL_GAP * 2,
        vmStartY:          0,
        subnetXPositions:  [],
        subnetInGroupY:    LAYOUT_CONFIG.BASE_PADDING,
        overlayInGroupY:   0,
        vmVerticalSpacing: 120,
      };

      let currentX = LAYOUT_CONFIG.BASE_PADDING * 2;

      subnetWidths.forEach((width) => {
        layout.subnetXPositions.push(currentX);
        currentX += width + LAYOUT_CONFIG.HORIZONTAL_GAP * 3;
      });

      const hasAnyOverlay = Object.values(subnetDimensionsCache).some(
        (d) => d.hasOverlay,
      );
      const estimatedSubnetAreaHeight = hasAnyOverlay ? 350 : 150;

      layout.vmStartY =
        layout.subnetY +
        estimatedSubnetAreaHeight +
        LAYOUT_CONFIG.VERTICAL_GAP * 2;

      return layout;
    },

    buildSubnetNodes(
      nodes,
      edges,
      subnet,
      dims,
      startX,
      layout,
      providerDisplay,
    ) {
      const sName = subnet.metadata.name;
      const sId = `subnet-${ sName }`;

      if (dims.hasOverlay) {
        const gId = `group-${ sName }`;

        // Group Container
        nodes.push({
          id:       gId,
          type:     'group',
          position: {
            x: startX,
            y: layout.subnetY - LAYOUT_CONFIG.BASE_PADDING,
          },
          data:  { type: 'group' },
          style: {
            background:   'rgba(0,0,0,0.05)',
            borderWidth:  '0px',
            borderRadius: '16px',
            padding:      `${ LAYOUT_CONFIG.BASE_PADDING }px`,
            width:        `${ dims.groupWidth }px`,
            height:       `${ dims.groupHeight }px`,
          },
          zIndex:     -1,
          selectable: false,
        });

        // Center Subnet in Group
        const subnetRelativeX = (dims.groupWidth - dims.subnetWidth) / 2;

        nodes.push({
          id:         sId,
          type:       'subnet',
          parentNode: gId,
          extent:     'parent',
          position:   { x: subnetRelativeX, y: layout.subnetInGroupY },
          data:       {
            type:     'subnet',
            name:     sName,
            cidr:     subnet.spec?.cidrBlock,
            provider: providerDisplay,
          },
          style: { width: `${ dims.subnetWidth }px` },
        });

        // Center Overlay in Group
        const overlayNodeId = `overlay-${ subnet.spec?.provider?.replace(
          /\//g,
          '-',
        ) }`;
        const overlayRelativeX = (dims.groupWidth - dims.overlayWidth) / 2;
        const overlayRelativeY =
          layout.subnetInGroupY +
          dims.subnetHeight +
          LAYOUT_CONFIG.VERTICAL_GAP;

        nodes.push({
          id:         overlayNodeId,
          type:       'overlay-network',
          parentNode: gId,
          extent:     'parent',
          position:   { x: overlayRelativeX, y: overlayRelativeY },
          data:       {
            type:           'overlay-network',
            name:           dims.nadName,
            nadType:        dims.nadType,
            clusterNetwork: dims.nadClusterNetwork,
            subnetId:       sId,
          },
          style: { width: `${ dims.overlayWidth }px` },
        });

        // Internal Group Edge (Static, Dashed, Gray)
        edges.push({
          id:       `edge-subnet-overlay-${ sName }`,
          source:   sId,
          target:   overlayNodeId,
          type:     'straight',
          animated: false,
          style:    {
            stroke:          COLORS.LINK_GRAY,
            strokeWidth:     2,
            strokeDasharray: '4,4',
          },
        });
      } else {
        // Standalone Subnet
        nodes.push({
          id:       sId,
          type:     'subnet',
          position: { x: startX, y: layout.subnetY },
          data:     {
            type:     'subnet',
            name:     sName,
            cidr:     subnet.spec?.cidrBlock,
            provider: providerDisplay,
          },
          style: { width: `${ dims.subnetWidth }px` },
        });
      }

      // Edge from VPC to Subnet
      edges.push({
        id:       `edge-vpc-${ sName }`,
        source:   'vpc',
        target:   sId,
        animated: true,
        style:    { stroke: COLORS.VPC, strokeWidth: 2 },
      });
    },

    buildVMNodes(
      nodes,
      edges,
      vpcVMs,
      vmToSubnets,
      vmToIps,
      vmToInterfaceDetails,
      layout,
      vpcSubnets,
    ) {
      const processedVMs = new Set();
      const subnetLayoutTrackers = {};

      vpcSubnets.forEach((subnet, index) => {
        const sName = subnet.metadata.name;

        subnetLayoutTrackers[sName] = {
          currentX:
            layout.subnetXPositions[index] + LAYOUT_CONFIG.BASE_PADDING * 2,
          currentRow: 0,
          vmsInRow:   0,
        };

        const vmList = vpcVMs.filter((vm) => (vmToSubnets[vm.metadata?.name] || []).includes(sName),
        );
        const singleNetworkVMs = vmList.filter(
          (vm) => (vmToSubnets[vm.metadata?.name] || []).length === 1,
        );
        const multiNetworkVMs = vmList.filter(
          (vm) => (vmToSubnets[vm.metadata?.name] || []).length > 1 &&
            !processedVMs.has(vm.metadata?.name),
        );

        const createNode = (vm, isMulti) => {
          const vmName = vm.metadata?.name || 'Unknown';

          if (isMulti) processedVMs.add(vmName);

          const vmId = `vm-${ vmName }`;
          const vmStatus = vm.status?.printableStatus || 'Unknown';
          const isStopped = ['stopped', 'off', 'paused'].includes(
            vmStatus.toLowerCase(),
          );
          const statusText = isStopped ? ` (${ vmStatus })` : '';

          const interfaces =
            (vmToInterfaceDetails[vmName] || []).length > 0 ? vmToInterfaceDetails[vmName] : (vmToIps[vmName] || []).map((ip) => ({
              mac: DEFAULT_VALUES.MAC,
              ip,
            }));

          const { width: vmWidth } = this.calculateVMDimensions(
            vmName,
            statusText,
            interfaces,
          );
          const tracker = subnetLayoutTrackers[sName];

          if (tracker.vmsInRow >= LAYOUT_CONFIG.VMS_PER_ROW) {
            tracker.currentRow++;
            tracker.currentX =
              layout.subnetXPositions[index] + LAYOUT_CONFIG.BASE_PADDING * 2;
            tracker.vmsInRow = 0;
          }

          nodes.push({
            id:       vmId,
            type:     isMulti ? 'multi-network-vm' : 'vm',
            position: {
              x: tracker.currentX,
              y:
                layout.vmStartY + tracker.currentRow * layout.vmVerticalSpacing,
            },
            data: {
              type:    isMulti ? 'multi-network-vm' : 'vm',
              name:    vmName,
              status:  vmStatus,
              isStopped,
              interfaces,
              subnets: vmToSubnets[vmName],
            },
            style:  { width: `${ vmWidth }px` },
            zIndex: 10,
          });

          tracker.currentX += vmWidth + LAYOUT_CONFIG.HORIZONTAL_GAP;
          tracker.vmsInRow++;

          const targetSubnets = isMulti ? vmToSubnets[vmName] : [sName];

          targetSubnets.forEach((targetSubnetName) => {
            const targetSubnetObj = vpcSubnets.find(
              (s) => s.metadata.name === targetSubnetName,
            );
            const provider = targetSubnetObj?.spec?.provider || PROVIDERS.OVN;
            const hasOverlay =
              provider && provider !== PROVIDERS.OVN && provider.trim() !== '';

            const sourceNodeId = hasOverlay ? `overlay-${ provider.replace(/\//g, '-') }` : `subnet-${ targetSubnetName }`;

            edges.push({
              id:       `edge-${ targetSubnetName }-${ vmId }`,
              source:   sourceNodeId,
              target:   vmId,
              animated: !isStopped,
              style:    {
                stroke:          isStopped ? COLORS.STOPPED : COLORS.VM,
                strokeWidth:     1,
                strokeDasharray: '5,5',
                opacity:         isStopped ? 0.4 : 1,
              },
            });
          });
        };

        singleNetworkVMs.forEach((vm) => createNode(vm, false));
        multiNetworkVMs.forEach((vm) => createNode(vm, true));
      });
    },

    // ==========================================
    // 4. Layout Adjustment
    // ==========================================

    resizeGroups() {
      const GAP = LAYOUT_CONFIG.VERTICAL_GAP;
      const PADDING = LAYOUT_CONFIG.BASE_PADDING;

      this.nodes.forEach((node) => {
        if (node.type === 'group') {
          const subnetNode = this.nodes.find(
            (n) => n.parentNode === node.id && n.data?.type === 'subnet',
          );
          const overlayNode = this.nodes.find(
            (n) => n.parentNode === node.id && n.data?.type === 'overlay-network',
          );

          if (subnetNode && overlayNode) {
            const subnetEl = document.querySelector(
              `[data-id="${ subnetNode.id }"] .custom-node`,
            );
            const overlayEl = document.querySelector(
              `[data-id="${ overlayNode.id }"] .custom-node`,
            );

            if (subnetEl && overlayEl) {
              const sW = subnetEl.offsetWidth;
              const sH = subnetEl.offsetHeight;
              const oW = overlayEl.offsetWidth;
              const oH = overlayEl.offsetHeight;

              const maxContentWidth = Math.max(sW, oW);
              const groupWidth = maxContentWidth + PADDING * 2;
              const groupHeight = PADDING + sH + GAP + oH + PADDING;

              node.style.width = `${ groupWidth }px`;
              node.style.height = `${ groupHeight }px`;

              subnetNode.position.x = (groupWidth - sW) / 2;
              overlayNode.position.x = (groupWidth - oW) / 2;

              subnetNode.position.y = PADDING;
              overlayNode.position.y = PADDING + sH + GAP;
            }
          }
        }
      });
    },

    // ==========================================
    // 5. Interaction (Highlighting)
    // ==========================================

    onNodeClick(event) {
      const { node } = event;

      if (node.data?.type === 'group') return;

      if (this.selectedNodeId === node.id) {
        this.selectedNodeId = null;
        this.relatedIds = new Set();
      } else {
        this.selectedNodeId = node.id;
        this.findRelatedNodes(node.id);
      }
    },

    findRelatedNodes(startNodeId) {
      const related = new Set([startNodeId]);

      // 1. Find Descendants (Downstream)
      const findDescendants = (currentId) => {
        this.edges.forEach((edge) => {
          if (edge.source === currentId && !related.has(edge.target)) {
            related.add(edge.target);
            findDescendants(edge.target);
          }
        });
      };

      // 2. Find Ancestors (Upstream)
      const findAncestors = (currentId) => {
        this.edges.forEach((edge) => {
          if (edge.target === currentId && !related.has(edge.source)) {
            related.add(edge.source);
            findAncestors(edge.source);
          }
        });
      };

      findDescendants(startNodeId);
      findAncestors(startNodeId);

      this.relatedIds = related;
    },

    onPaneClick() {
      this.selectedNodeId = null;
      this.relatedIds = new Set();
    },

    getCssVar(name) {
      return getComputedStyle(document.body).getPropertyValue(name).trim();
    },
  },
};
</script>

<template>
  <div class="vpc-topology">
    <div class="topology-header">
      <div class="visibility-controls">
        <Checkbox
          v-for="option in visibilityOptions"
          :key="option.modelKey"
          v-model:value="$data[option.modelKey]"
          class="control-item"
          :label="option.label"
          :disabled="option.disabled"
        >
          <template #label>
            {{ option.label }}
            <span
              class="count-badge"
              :style="{
                background: colors[option.badgeColorKey],
                opacity: option.disabled ? 0.5 : 1,
              }"
            >{{ option.count }}</span>
          </template>
        </Checkbox>
      </div>
    </div>

    <div
      v-if="loading"
      class="loading"
    >
      <i class="icon icon-spinner icon-spin" /> Loading topology...
    </div>

    <div
      v-else-if="nodes.length === 0"
      class="empty-state"
    >
      <i class="icon icon-info" />
      <p>No resources found</p>
    </div>

    <VueFlow
      v-else
      :nodes="filteredNodes"
      :edges="filteredEdges"
      class="vpc-flow"
      :default-zoom="0.8"
      :min-zoom="0.2"
      :max-zoom="2"
      :pan-on-drag="true"
      fit-view-on-init
      @node-click="onNodeClick"
      @pane-click="onPaneClick"
    >
      <template #node-vpc="{ data }">
        <Handle
          type="source"
          position="bottom"
        />
        <div
          class="custom-node vpc-node"
          :class="data.stateClass"
          :style="{
            width: layoutConfig.VPC_WIDTH + 'px',
            border: `2px solid ${colors.VPC}`,
          }"
        >
          <div class="node-name">
            {{ data.name }}
          </div>
        </div>
      </template>

      <template #node-group></template>

      <template #node-subnet="{ data }">
        <Handle
          type="target"
          position="top"
          style="left: 50%; transform: translate(-50%, -50%)"
        />
        <Handle
          type="source"
          position="bottom"
          style="left: 50%; transform: translate(-50%, 50%)"
        />
        <div
          class="custom-node subnet-node"
          :class="data.stateClass"
          :style="{ width: '100%', border: `2px solid ${colors.SUBNET}` }"
        >
          <div class="node-name">
            {{ data.name }}
          </div>
          <div class="node-details">
            <div>CIDR: {{ data.cidr }}</div>
            <div>Provider: {{ data.provider }}</div>
          </div>
        </div>
      </template>

      <template #node-overlay-network="{ data }">
        <Handle
          type="target"
          position="top"
          style="left: 50%; transform: translate(-50%, -50%)"
        />
        <Handle
          type="source"
          position="bottom"
          style="left: 50%; transform: translate(-50%, 50%)"
        />
        <div
          class="custom-node overlay-node"
          :class="data.stateClass"
          :style="{ width: '100%', border: `2px dashed ${colors.OVERLAY}` }"
        >
          <div class="node-name">
            {{ data.name }}
          </div>
          <div class="node-details">
            <div>Cluster Network: {{ data.clusterNetwork }}</div>
          </div>
        </div>
      </template>

      <template #node-vm="{ data }">
        <Handle
          type="target"
          position="top"
        />
        <div
          class="custom-node vm-node"
          :class="[data.stateClass, { stopped: data.isStopped }]"
          :style="{
            width: layoutConfig.VM_WIDTH + 'px',
            border: data.isStopped
              ? `2px dashed ${colors.STOPPED}`
              : `2px solid ${colors.VM}`,
            backgroundColor: data.isStopped ? colors.BG_STOP : '#ffffff',
          }"
        >
          <div
            class="node-name"
            :style="{ color: data.isStopped ? colors.MUTED : colors.TEXT }"
          >
            {{ data.name }}{{ data.isStopped ? ` (${data.status})` : "" }}
          </div>
          <div
            class="node-details"
            :style="{ color: data.isStopped ? colors.MUTED : colors.TEXT }"
          >
            <div
              v-for="(iface, idx) in data.interfaces"
              :key="idx"
              class="interface-group"
            >
              <div>IP: {{ iface.ip }}</div>
              <div class="mac-text">
                MAC: {{ iface.mac }}
              </div>
            </div>
          </div>
        </div>
      </template>

      <template #node-multi-network-vm="{ data }">
        <Handle
          type="target"
          position="top"
        />
        <div
          class="custom-node vm-node"
          :class="[data.stateClass, { stopped: data.isStopped }]"
          :style="{
            width: layoutConfig.VM_WIDTH + 'px',
            border: data.isStopped
              ? `2px dashed ${colors.STOPPED}`
              : `2px solid ${colors.VM}`,
            backgroundColor: data.isStopped ? colors.BG_STOP : '#ffffff',
          }"
        >
          <div
            class="node-name"
            :style="{ color: data.isStopped ? colors.MUTED : colors.TEXT }"
          >
            {{ data.name }}{{ data.isStopped ? ` (${data.status})` : "" }}
          </div>
          <div
            class="node-details"
            :style="{ color: data.isStopped ? colors.MUTED : colors.TEXT }"
          >
            <div
              v-for="(iface, idx) in data.interfaces"
              :key="idx"
              class="interface-group"
            >
              <div>IP: {{ iface.ip }}</div>
              <div class="mac-text">
                MAC: {{ iface.mac }}
              </div>
            </div>
          </div>
        </div>
      </template>

      <Background
        :pattern-color="backgroundPatternColor"
        :gap="12"
        size="1"
      />
      <Controls /><MiniMap />
    </VueFlow>
  </div>
</template>

<style lang="scss" scoped>
$transition-duration: 0.5s;
$transition-ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
$transition-ease-smooth: cubic-bezier(0.4, 0, 0.2, 1);

.vpc-topology {
  height: 800px;
  width: 100%;
  background: var(--body-bg);
  border-radius: 4px;
  overflow: hidden;

  .topology-header {
    display: flex;
    align-items: center;
    padding: 12px 0;
    border-bottom: 1px solid var(--border);
    .visibility-controls {
      display: flex;
      gap: 24px;
      align-items: center;
      .control-item {
        display: flex;
        align-items: center;
        cursor: pointer;
        user-select: none;

        ::v-deep .checkbox-label {
          display: flex;
          align-items: center;
          line-height: 1;
        }

        .count-badge {
          display: inline-flex;
          align-items: center;
          justify-content: center;
          padding: 2px 8px;
          border-radius: 30px;
          font-size: 12px;
          min-width: 20px;
          height: 20px;
          text-align: center;
          color: #ffffff;
          margin-left: 8px;
          line-height: 1;
        }
      }
    }
  }

  .loading {
    display: flex;
    align-items: center;
    justify-content: center;
    height: calc(100% - 100px);
    font-size: 16px;
    color: var(--muted);
    i {
      margin-right: 10px;
      font-size: 20px;
    }
  }

  .empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: calc(100% - 100px);
    color: var(--muted);
    i {
      font-size: 48px;
      margin-bottom: 16px;
    }
    p {
      font-size: 16px;
      margin: 0;
    }
  }

  .vpc-flow {
    height: calc(100% - 100px);
  }

  .vue-flow__node {
    transition: all $transition-duration $transition-ease-spring;
  }

  ::v-deep(.vue-flow__edge) {
    transition: opacity $transition-duration $transition-ease-smooth,
      filter $transition-duration $transition-ease-smooth;
    path {
      transition: stroke-dasharray 0.4s ease;
    }
    &.dimmed {
      opacity: 0.05 !important;
      filter: grayscale(90%);
      path {
        stroke-dasharray: 5 !important;
      }
    }
  }

  .custom-node {
    padding: 10px;
    font-size: 13px;
    line-height: 1.4;
    color: #000000;
    box-sizing: border-box;
    background: #ffffff;
    border-radius: 12px;
    height: auto;

    transition: transform $transition-duration $transition-ease-spring,
      opacity $transition-duration $transition-ease-smooth,
      box-shadow $transition-duration $transition-ease-smooth,
      filter $transition-duration $transition-ease-smooth,
      background-color $transition-duration $transition-ease-smooth;

    opacity: 1;
    filter: grayscale(0%);
    transform: scale(1);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);

    .node-name {
      font-weight: 700;
      font-size: 18px;
      margin-bottom: 4px;
    }
    .node-details {
      font-size: 14px;
      .interface-group {
        &:not(:first-child) {
          margin-top: 6px;
          padding-top: 6px;
        }
      }
    }

    &.node-focused {
      transform: scale(1.12) translateY(-4px);
      z-index: 1000;
      opacity: 1 !important;
      box-shadow: 0 15px 35px -5px rgba(0, 0, 0, 0.15),
        0 5px 15px -3px rgba(0, 0, 0, 0.08), 0 0 0 3px rgba(36, 83, 255, 0.15),
        0 0 20px 2px rgba(36, 83, 255, 0.1);
      background-color: #fafafa;
    }

    &.node-related {
      transform: scale(1.03) translateY(-1px);
      z-index: 999;
      opacity: 1 !important;
      box-shadow: 0 8px 20px -3px rgba(0, 0, 0, 0.08),
        0 3px 8px -2px rgba(0, 0, 0, 0.04);
    }

    &.node-dimmed {
      opacity: 0.3;
      filter: grayscale(85%) brightness(95%);
      transform: scale(0.98);
    }
  }

  .subnet-node.node-focused {
    box-shadow: 0 15px 35px -5px rgba(0, 0, 0, 0.15),
      0 0 0 3px rgba(254, 124, 63, 0.15), 0 0 20px 2px rgba(254, 124, 63, 0.1);
  }

  .vm-node.node-focused {
    transform: scale(1.12) translateY(-6px);
    box-shadow: 0 20px 40px -5px rgba(0, 0, 0, 0.18),
      0 0 0 3px rgba(0, 189, 167, 0.2), 0 0 25px 3px rgba(0, 189, 167, 0.15);
  }

  .vpc-node {
    border: 2px solid #2453ff;
    cursor: pointer;
  }
  .subnet-node {
    border: 2px solid #fe7c3f;
    cursor: pointer;
  }
  .overlay-node {
    font-size: 11px;
    border: 2px dashed #9333ea;
  }
  .vm-node {
    font-size: 11px;
    border: 2px solid #00bda7;
    &.stopped {
      background: #f3f4f6;
      color: #9ca3af;
      border: 2px dashed #9ca3af;
      opacity: 0.6;
    }
  }
}
</style>
