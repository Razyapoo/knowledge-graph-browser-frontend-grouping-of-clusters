<template>
	<div class="d-flex flex-grow-1 wrapper" ref="wrapper">
		<div class="flex-grow-1 graph-area" ref="graphd"></div>

		<v-toolbar flat dense floating class="ma-3 toolbar" :style="leftStyle">
			<search-component :graph-searcher="graphSearcher" @searched="manipulator.locateOrTryFetchNode($event)"></search-component>
		</v-toolbar>
		<v-main v-if="layoutManager.currentLayout.constraintRulesLoaded && layoutManager.currentLayout.supportsHierarchicalView" :title="$t('hierarchy.hint')" class="checkbox" :disabled="modeCompact" :style="rightStyle">
			<v-container>
				<p>{{ $tc('zooming.zooming_options', 1) }}</p>
				<v-main :title="$t('zooming.global_grouping_hint')">
					<v-checkbox
						class="ma-0 pa-0"
						:label="$tc('zooming.global_grouping', 1)"
						
						@click = "groupingOfClustersEvent(true)"
					></v-checkbox>
				</v-main>
				<v-main :title="$t('zooming.local_grouping_hint')">
					<v-checkbox
						class="ma-0 pa-0"
						:label="$tc('zooming.local_grouping', 1)"
						:title="$t('zooming.local_grouping_hint')"
						@click = "groupingOfClustersEvent(false)"
					></v-checkbox>
				</v-main>
				<v-main title="">
					<v-checkbox
						class="ma-0 pa-0"
						:label="$tc('zooming.zooming', 1)"
						:input-value="true"
						@click = "zoomingEvent()"
					></v-checkbox>
				</v-main>
			</v-container>
		</v-main>
		<div class="my-3 mx-5 buttons v-toolbar" :style="rightStyle">
			<div class="my-2">
				<v-btn color="primary" fab small :dark="!modeCompact" :disabled="modeCompact" @click="areaManipulator.zoomIn()">
					<v-icon>{{ icons.zoomIn }}</v-icon>
				</v-btn>
			</div>
			<div class="my-2">
				<v-btn color="primary" fab small :dark="!modeCompact" :disabled="modeCompact" @click="areaManipulator.zoomOut()">
					<v-icon>{{ icons.zoomOut }}</v-icon>
				</v-btn>
			</div>
			<div class="my-2">
				<v-btn color="primary" fab small :dark="!modeCompact" :disabled="modeCompact" @click="areaManipulator.fit()">
					<v-icon>{{ icons.fit }}</v-icon>
				</v-btn>
			</div>
			<div class="my-2" v-if="layoutManager.currentLayout.supportsCompactMode">
				<v-btn color="primary" fab small :dark="isNodeSelected" :disabled="!isNodeSelected" @click="compactModeChange(!modeCompact)">
					<v-icon>{{ icons.compactMode[modeCompact ? 1 : 0] }}</v-icon>
				</v-btn>
			</div>
			<component v-if="layoutManager.currentLayoutData.buttons" :is="layoutManager.currentLayoutData.buttons" :layout="layoutManager.currentLayout" />
		</div>

		<graph-element-node
			v-for="node in graph.nodes"
			v-if="node.mounted && !node.belongsToGroup"
			:key="node.IRI.replace(/\./, '_')"
			:node="node"
			:areaManipulator="areaManipulator"
			:node-locking-supported="layoutManager.currentLayout.supportsNodeLocking"
			:explicitly-active="!isNodeSelected"
			:mode-compact="modeCompact"
		/>
		<graph-element-node-group
			v-for="group in graph.groups"
			v-if="group.mounted"
			:key="group.id"
			:node="group"
			:manipulator="manipulator"
			:areaManipulator="areaManipulator"
			:node-locking-supported="layoutManager.currentLayout.supportsNodeLocking"
			:explicitly-active="!isNodeSelected"
			:mode-compact="modeCompact"
		/>
		<graph-element-edge
			v-for="edge in graph.edges"
			v-if="edge.source.mounted && edge.target.mounted && !edge.source.belongsToGroup && !edge.target.belongsToGroup"
			:key="edge.identifier.replace(/\./, '_')"
			:edge="edge"
			:explicitly-active="!isNodeSelected"
            :mode-compact="modeCompact"
		/>
		<graph-element-edge
			v-for="edge in groupEdges"
			v-if="edge.source.mounted && edge.target.mounted"
			:key="edge.identifier.replace(/\./, '_')"
			:edge="edge"
			:explicitly-active="!isNodeSelected"
			:mode-compact="modeCompact"
		/>
	</div>
</template>

<script lang="ts">
import Component from "vue-class-component";
import GraphElementNode from "./GraphElementNode.vue";
import GraphElementEdge from "./GraphElementEdge";
import Cytoscape from "cytoscape";
import {Emit, Mixins, Prop, Watch} from "vue-property-decorator";
import {ResponseStylesheet} from "../../remote-server/ResponseInterfaces";
import {Graph} from "../../graph/Graph";
import {mdiPlus, mdiMinus, mdiArrowExpandAll, mdiChartBubble, mdiArrowDecisionOutline} from '@mdi/js';
import SearchComponent from "../SearchComponent.vue";
import GraphAreaManipulator from "../../graph/GraphAreaManipulator";
import ViewOptions from "../../graph/ViewOptions";
import GraphSearcher from "../../searcher/GraphSearcher";
import GraphManipulator from "../../graph/GraphManipulator";
import GraphAreaStylesheetMixin from "./GraphAreaStylesheetMixin";

import cola from 'cytoscape-cola';
import popper from 'cytoscape-popper';
import {LayoutManager} from "../../layout/LayoutManager";
import GraphElementNodeGroup from "./GraphElementNodeGroup.vue";
import GroupEdge from "../../graph/GroupEdge";
import NodeCommon from "../../graph/NodeCommon";
import EdgeCommon from "../../graph/EdgeCommon";
import NodeGroup from "@/graph/NodeGroup";
import Vue from "vue";

const WHEEL_SENSITIVITY = 0.2;

@Component({
	components: {
		SearchComponent,
		GraphElementNode,
		GraphElementEdge,
		GraphElementNodeGroup,
	}
})
export default class GraphArea extends Mixins(GraphAreaStylesheetMixin) {
	@Prop() graph: Graph;
	@Prop() stylesheet: ResponseStylesheet;
	@Prop() leftOffset: number;
	@Prop() rightOffset: number;
	@Prop() viewOptions: ViewOptions;
	@Prop() private graphSearcher: GraphSearcher;
	@Prop() manipulator: GraphManipulator;
	@Prop(Object) private areaManipulator: GraphAreaManipulator;
	@Prop(Object) private layoutManager: LayoutManager;

	/**
	 * Compact mode is a mode where selected nodes with all its neighbours are layouted independently of others
	 * */
	@Prop(Boolean) private modeCompact !: boolean;
	@Prop(Boolean) private groupModeCompact !: boolean;

	groupCompactNodesBuffer: NodeCommon[] = [];
	/**
	 * How much of the graph area is covered by panels. This array is readonly so it could be passed by reference.
	 * top, right, bottom, left
	 * Currently only right and left is supported
	 * @readonly
	 */
	private readonly offset: [number, number, number, number] = [0, 0, 0, 0];

	private icons = {
		zoomIn: mdiPlus,
		zoomOut: mdiMinus,
		fit: mdiArrowExpandAll,
		compactMode: [mdiArrowDecisionOutline, mdiChartBubble]
	}

	/**
	 * Cytoscape instance
	 * @non-reactive
	 */
	cy !: Cytoscape.Core;

	get leftStyle(): string {
		return 'left: ' + this.leftOffset + 'px;';
	}

	get rightStyle(): string {
		return 'right: ' + this.rightOffset + 'px;';
	}

	@Emit()
	newManipulator() {
		let manipulator = new GraphAreaManipulator(this.cy, this.offset);
		manipulator.graphArea = this;
		return manipulator;
	}

	/**
	 * From all groups returns edges
	 */
	get groupEdges(): GroupEdge[] {
		let edges: GroupEdge[] = [];

		for (let group of this.graph.groups) {
			if (group.mounted) {
				edges = [...edges, ...group.visibleGroupEdges];
			}
		}

		return edges;
	}

	/**
	 * Called by Vue framework
	 */
	created() {
		Cytoscape.use(cola);
		Cytoscape.use(popper);

		this.cy = Cytoscape({
			wheelSensitivity: WHEEL_SENSITIVITY
    	});

		this.stylesheetUpdated();
	}

	public mountToElement() {
		this.cy = Cytoscape({
			wheelSensitivity: WHEEL_SENSITIVITY
		});
		this.stylesheetUpdated();
		this.cy.mount(<Element>this.$refs.graphd);
	}

	//#region Cytoscape batch optimisation
	cyBatchInProcessOptimisation: boolean = false;

	beforeUpdate() {
		if (!this.cyBatchInProcessOptimisation && this.areaManipulator.cy) {
			this.areaManipulator.cy.startBatch();
			this.cyBatchInProcessOptimisation = true;
		}
	}

	updated() {
		if (this.cyBatchInProcessOptimisation) {
			this.areaManipulator.cy.endBatch();
			this.cyBatchInProcessOptimisation = false;
		}
	}
	//#endregion Cytoscape batch optimisation

	// #region Tracking of chosen options of clustering checkbox
	
	/** 
	 * Handles zooming event \
	 * For more information see https://github.com/Razyapoo/KGBClusteringDocumentation/blob/main/user_documentation.md#checkbox-1 
	 */
	public zoomingEvent() {
		
		if (this.areaManipulator.isZoomingChecked) {
			this.areaManipulator.isZoomingChecked = false;
			this.cy.zoomingEnabled(false);
		}
		else {
			this.areaManipulator.isZoomingChecked = true;
			this.cy.zoomingEnabled(true);
		}	

	}

	/** 
	 * Handles grouping of clusters event \
	 * For more information see https://github.com/Razyapoo/KGBClusteringDocumentation/blob/main/user_documentation.md#checkbox-1 
	 */
	public groupingOfClustersEvent(global: boolean = true) {
		if (global) {
			if (this.areaManipulator.isGlobalGroupingOfClustersChecked) {
				this.areaManipulator.isGlobalGroupingOfClustersChecked = false;
			}
			else {
				this.areaManipulator.isGlobalGroupingOfClustersChecked = true;
			}	
		} else {
			if (this.areaManipulator.isLocalGroupingOfClustersChecked) {
				this.areaManipulator.isLocalGroupingOfClustersChecked = false;
			}
			else {
				this.areaManipulator.isLocalGroupingOfClustersChecked = true;
			}	
		}

	}
	// #endregion

	@Watch('modeCompact')
	modeCompactChanged() {
		this.cy.userPanningEnabled(!this.modeCompact);
		this.cy.userZoomingEnabled(!this.modeCompact);
		this.cy.boxSelectionEnabled(!this.modeCompact);

		if (!this.modeCompact) {
			let rootGroups = this.graph.groups.filter(group => !group.groupCompactParent && (group.groupCompactChildren.length > 0));
			rootGroups.forEach(group => this.compactUnlockRecursively(group));
		}
	}

    @Watch('dataForCompactMode')
    private dataForCompactModeChanged() {
		let [nodes, edges] = this.dataForCompactMode;

	    if (nodes && !nodes.length) {
	    	this.compactModeChange(false);
	    	return;
		}

        this.layoutManager.currentLayout.onCompactMode(nodes, edges);

        if (nodes) {
			this.areaManipulator.fitFollowSet(nodes);
		} else {
        	this.areaManipulator.fitFollowStop();
		}
    }

    @Watch('layoutManager.currentLayout')
    private currentLayoutChanged() {
		this.compactModeChange(false);
	}

    /**
     * This getter returns which nodes and edges are in compact mode. If compact mode is turned off, the return values
     * are [null, null].
     * */
    private get dataForCompactMode(): [NodeCommon[], EdgeCommon[]] {
	    if (this.modeCompact) {

	        // let nodes: NodeCommon[] = this.graph.nodesVisual.filter(node => (node.selected || node.neighbourSelected) && node.element);
	        // // let nodes: NodeCommon[] = this.graph.nodesVisual.filter(node => (node.selected) && node.element);

	        // let edges: EdgeCommon[] = this.graph.edgesVisual.filter(edge => (edge.source.selected || edge.target.selected) && edge.element);
			
			
			// if (this.layoutManager.currentLayout.supportsHierarchicalView && this.layoutManager.currentLayout.constraintRulesLoaded) {
			// 	// let nodesForCompact: NodeCommon[] = [];
			// 	let addChildrenRecursively = function(node: NodeCommon) {
			// 		if (node.children?.length > 0) {
			// 			node.children.forEach(child => {
			// 				if (child.mounted && !child.isUnmountedAndHiddenInHierarchy) {
			// 					if (!nodes.find(nd => nd == child)) nodes.push(child);
			// 					addChildrenRecursively(child);
			// 				}
			// 			});
			// 		} 
			// 	}
						
			// 	nodes.forEach(node => addChildrenRecursively(node));
			// }

			// return [nodes, edges];
			
			let groups: NodeGroup[] = this.graph.groups.filter(group => (group.selected) && group.element);
			if (groups.length > 0) {
				let nodes: NodeCommon[] = [];
				groups.forEach(group => {
					if (group.groupCompactParent)  {
						group.groupCompactParent.groupCompactChildren.forEach(innerGroup => {
							if (innerGroup != group && innerGroup instanceof NodeGroup) this.compactUnlockRecursively(innerGroup);
						})
					}
					if (group.groupCompactChildren.length > 0) {
						group.groupCompactChildren.forEach(child =>{
							if (child instanceof NodeGroup) this.compactUnlockRecursively(child);	
						})
					} else {
						group.nodes.forEach(node => { 
							// this.groupCompactNodesBuffer.push(node); 
							group.groupCompactChildren.push(node);
							nodes.push(node); 
							node.mounted = true; 
							node.groupCompactBelongsToGroupCache = node.belongsToGroup; 
							node.belongsToGroup = null;
						});
					}

					// Make a group as new "pseudo-parent", trick here is that after degroup, node will be mounted 
					// and registered again with correct parent, because it will inherit parent from group
					Vue.nextTick(() => {
						for (let child of group.nodes) {
							if (child.element) {
								// we can safely set new parent (group) both for element and for object
								let parent = this.areaManipulator.cy.getElementById(group.identifier).id();
								this.areaManipulator.cy.getElementById(child.identifier).move({
									parent: parent
								});
								child.groupCompactParent = group;
							}
						}
					});
				})
	
				this.areaManipulator.layoutManager.currentLayout.onGroupChangedCompact();
	
				return [[...nodes, ...groups], null];
			}
        }

		return [null, null];

    }

	// Collapse all "pseudo-child" of group again into group
	private compactUnlockRecursively(group: NodeGroup) {
		if (group.groupCompactChildren.length > 0) {
			group.groupCompactChildren.forEach(child => {
				if (child instanceof NodeGroup) this.compactUnlockRecursively(child);
				child.belongsToGroup = child.groupCompactBelongsToGroupCache;
				child.groupCompactBelongsToGroupCache = null;
				child.groupCompactParent = null;
				child.selected = false;
				if (child instanceof NodeGroup) child.mounted = false;
			})
			group.groupCompactChildren = [];
		}
	}
	
    @Emit()
	private compactModeChange(val: boolean) { 
		return val; 
	}

	/**
	 * Computes whether there is at least one node in the graph which is selected and visible.
	 */
	private get isNodeSelected(): boolean {
		for (let iri in this.graph.nodes) {
			let node = this.graph.nodes[iri];

			if (node.mounted && node.isVisible && node.selected && !node.belongsToGroup) {
				return true;
			}
		}

		for (let group of this.graph.groups) {
			if (group.mounted && group.selected) {
				return true;
			}
		}

		return false;
	}

	/**
	 * Vue method called after the creation of the object.
	 * Mounts the Cytoscape instance to HTML and registers basic events handlers
	 */
	mounted() {
		// Mount Cytoscape instance to HTML element
		this.cy.mount(<Element>this.$refs.graphd);
		//this.mountToElement();

		// Double-click handeling
		let doubleClickDelayMs = 350;
		let previousTapStamp = 0;
		this.cy.on('tap', 'node', (event) => {
			let currentTapStamp = event.timeStamp;
			let msFromLastTap = currentTapStamp - previousTapStamp;
			previousTapStamp = currentTapStamp;

			if (msFromLastTap < doubleClickDelayMs) {
				(<GraphElementNode>event.target.scratch("_component")).onDoubleClicked();
			}
		});

		// Mount manipulator
		this.newManipulator();
	}

	@Watch('leftOffset', {immediate: true})
	@Watch('rightOffset', {immediate: true})
	private offsetChanged() {
		this.offset[1] = this.rightOffset;
		this.offset[3] = this.leftOffset;
	}
}
</script>
<style lang="scss" scoped>
.graph-area {
    flex: auto;
	position: absolute;
	width: 100%;
	height: 100%;
}
.wrapper {
	position: relative;
}
.toolbar {
	position: absolute;
	left: 56px;
}
.toolbar.toolbar-move {
	left: 256px;
}

.checkbox {
	position: absolute;
	right: 80px;
}

.buttons {
	position: absolute;
	bottom: 0;
	box-shadow: none;
}
</style>
