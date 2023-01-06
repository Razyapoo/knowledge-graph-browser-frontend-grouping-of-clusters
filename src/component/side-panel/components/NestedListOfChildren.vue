<template>
    <div class="nested-list-of-children">
      <div>{{ label }}</div>
      <nested-list-of-children
        v-for="node in nodes" :key="node.identifier"
        :nodes="getNodes(node)" 
        :label="getLabel(node)"
        :level="level + 1"
      >
      </nested-list-of-children>
    </div>
  </template>


<script lang="ts">
    import {Component, Emit, Prop, Watch} from "vue-property-decorator";
    import Vue from "vue";
    import NodeCommon from "@/graph/NodeCommon";
    import NodeGroup from "@/graph/NodeGroup";
    import {Node} from "@/graph/Node";
    @Component({
        components: { NestedListOfChildren }
    })
    export default class NestedListOfChildren extends Vue {
        @Prop() nodes: NodeCommon[];
        // @Prop(Boolean) selected: boolean = false;
        @Prop() level: number;
        @Prop() label: string;

        @Emit('nodeSelected')
        private nodeSelected(node: NodeCommon) {
            return node;
        }

        private getLabel(node: NodeCommon) {
            if (node instanceof Node) return node.currentView?.preview ? node.currentView.preview.label : "-";
            // if (node instanceof NodeGroup) return node.mostFrequentType ? node.mostFrequentType.label + " (" + node.nodes.length + ")" : "-";
        }

        private isSelected(node: NodeCommon) {
            return node.selected;
        }

        private getNodes(node: NodeCommon) {
            if (node instanceof NodeGroup) {
                return node.nodes;
            } else if (node.children?.length > 0) {
                return node.children;
            }
        }


    }
</script>