<template>
    <div ref="item" class="vue-grid-layout" :style="mergedStyle">
        <slot></slot>
        <grid-item class="vue-grid-placeholder"
                   v-show="isDragging"
                   :x="placeholder.x"
                   :y="placeholder.y"
                   :w="placeholder.w"
                   :h="placeholder.h"
                   :i="placeholder.i"></grid-item>
    </div>
</template>
<style>
    .vue-grid-layout {
        position: relative;
        transition: height 200ms ease;
    }
</style>
<script>
    import Vue from 'vue';
    var elementResizeDetectorMaker = require("element-resize-detector");

    import {bottom, compact, getLayoutItem, moveElement, validateLayout, cloneLayout} from '../helpers/utils';
    import {getBreakpointFromWidth, getColsFromBreakpoint, findOrGenerateResponsiveLayout} from "../helpers/responsiveUtils";
    //var eventBus = require('./eventBus');

    import GridItem from './GridItem.vue'
    import {addWindowEventListener, removeWindowEventListener} from "../helpers/DOM";

    export default {
        name: "GridLayout",
        provide() {
            return {
                eventBus: null
            }
        },
        components: {
            GridItem,
        },
        props: {
            // If true, the container height swells and contracts to fit contents
            autoSize: {
                type: Boolean,
                default: true
            },
            colNum: {
                type: Number,
                default: 12
            },
            rowHeight: {
                type: Number,
                default: 150
            },
            maxRows: {
                type: Number,
                default: Infinity
            },
            margin: {
                type: Array,
                default: function () {
                    return [10, 10];
                }
            },
            isDraggable: {
                type: Boolean,
                default: true
            },
            isResizable: {
                type: Boolean,
                default: true
            },
            isMirrored: {
                type: Boolean,
                default: false
            },
            useCssTransforms: {
                type: Boolean,
                default: true
            },
            verticalCompact: {
                type: Boolean,
                default: true
            },
            layout: {
                type: Array,
                required: true,
            },
            responsive: {
                type: Boolean,
                default: false
            },
            breakpoints:{
                type: Object,
                default: function(){return{ lg: 1200, md: 996, sm: 768, xs: 480, xxs: 0 }}
            },
            cols:{
                type: Object,
                default: function(){return{ lg: 12, md: 10, sm: 6, xs: 4, xxs: 2 }},
            },
            validateLayoutFn: {
                type: Function,
                default: function() { return true },
            }
        },
        data: function () {
            return {
                width: null,
                mergedStyle: {},
                lastLayoutLength: 0,
                isDragging: false,
                placeholder: {
                    x: 0,
                    y: 0,
                    w: 0,
                    h: 0,
                    i: -1
                },
                layouts: {}, // array to store all layouts from different breakpoints
                lastBreakpoint: null, // store last active breakpoint
                originalLayout: null, // store original Layout
                localLayout: [],
            };
        },
        created () {
            const self = this;

            // Accessible refernces of functions for removing in beforeDestroy
            self.resizeEventHandler = function(eventType, i, x, y, h, w) {
                self.resizeEvent(eventType, i, x, y, h, w);
            };

            self.dragEventHandler = function(eventType, i, x, y, h, w) {
                self.dragEvent(eventType, i, x, y, h, w);
            };

            self._provided.eventBus = new Vue();
            self.eventBus = self._provided.eventBus;
            self.eventBus.$on('resizeEvent', self.resizeEventHandler);
            self.eventBus.$on('dragEvent', self.dragEventHandler);
            self.$emit('layout-created', self.localLayout);
        },
        beforeDestroy: function(){
            //Remove listeners
            this.eventBus.$off('resizeEvent', this.resizeEventHandler);
            this.eventBus.$off('dragEvent', this.dragEventHandler);
			this.eventBus.$destroy();
            removeWindowEventListener("resize", this.onWindowResize);
			this.erd.uninstall(this.$refs.item);
        },
        beforeMount: function() {
            this.$emit('layout-before-mount', this.localLayout);
        },
        mounted: function() {
            this.$emit('layout-mounted', this.localLayout);
            this.$nextTick(function () {
                validateLayout(this.localLayout);

                this.originalLayout = this.localLayout;
                const self = this;
                this.$nextTick(function() {
                    if (self.width === null) {
                        self.onWindowResize();

                        self.initResponsiveFeatures();

                        //self.width = self.$el.offsetWidth;
                        addWindowEventListener('resize', self.onWindowResize);
                    }
                    compact(self.localLayout, self.verticalCompact);

                    self.updateHeight();
                    self.$nextTick(function () {
                        this.erd = elementResizeDetectorMaker({
                            strategy: "scroll" //<- For ultra performance.
                        });
                        this.erd.listenTo(self.$refs.item, function () {
                            self.onWindowResize();
                        });
                    });

                    self.$emit('layout-ready', self.localLayout);
                });
            });
        },
        watch: {
            width: function () {
                this.$nextTick(function () {
                    //this.$broadcast("updateWidth", this.width);
                    this.eventBus.$emit("updateWidth", this.width);
                    this.updateHeight();
                });
            },
            layout: function (val) {
                this.localLayout = val.map(item => JSON.parse(JSON.stringify(item)));
            },
            localLayout: function (val) {
                if (this.validateLayoutFn && this.validateLayoutFn(val)) {
                    this.lastValidLayout = val.map(item => JSON.parse(JSON.stringify(item)));
                } else if (this.validateLayoutFn) {
                    this.localLayout.forEach((item) => {
                        const idx = this.lastValidLayout.findIndex(item2 => item.i === item2.i);
                        this.$set(this.localLayout, idx, Object.assign(item, this.lastValidLayout[idx]));
                    })
                }
                this.layoutUpdate();
            },
            colNum: function (val) {
                this.eventBus.$emit("setColNum", val);
            },
            rowHeight: function() {
                this.eventBus.$emit("setRowHeight", this.rowHeight);
            },
            isDraggable: function() {
                this.eventBus.$emit("setDraggable", this.isDraggable);
            },
            isResizable: function() {
                this.eventBus.$emit("setResizable", this.isResizable);
            },
            responsive() {
                if (!this.responsive) {
                    this.$emit('update:layout', this.originalLayout);
                    this.eventBus.$emit("setColNum", this.colNum);
                }
                this.onWindowResize();
            }
        },
        methods: {
            layoutUpdate() {
                if (this.localLayout !== undefined && this.originalLayout !== null) {
                    if (this.localLayout.length !== this.originalLayout.length) {
                        // console.log("### LAYOUT UPDATE!", this.localLayout.length, this.originalLayout.length);

                        let diff = this.findDifference(this.localLayout, this.originalLayout);
                        if (diff.length > 0){
                            // console.log(diff);
                            if (this.localLayout.length > this.originalLayout.length) {
                                this.originalLayout = this.originalLayout.concat(diff);
                            } else {
                                this.originalLayout = this.originalLayout.filter(obj => {
                                    return !diff.some(obj2 => {
                                        return obj.i === obj2.i;
                                    });
                                });
                            }
                        }

                        this.lastLayoutLength = this.localLayout.length;
                        this.initResponsiveFeatures();
                    }

                    compact(this.localLayout, this.verticalCompact);
                    this.eventBus.$emit("updateWidth", this.width);
                    this.updateHeight();
                }
            },
            updateHeight: function () {
                this.mergedStyle = {
                    height: this.containerHeight()
                };
            },
            onWindowResize: function () {
                if (this.$refs !== null && this.$refs.item !== null && this.$refs.item !== undefined) {
                    this.width = this.$refs.item.offsetWidth;
                }
                this.eventBus.$emit("resizeEvent");
            },
            containerHeight: function () {
                if (!this.autoSize) return;
                return bottom(this.localLayout) * (this.rowHeight + this.margin[1]) + this.margin[1] + 'px';
            },
            dragEvent: function (eventName, id, x, y, h, w) {
                //console.log(eventName + " id=" + id + ", x=" + x + ", y=" + y);
                let l = getLayoutItem(this.localLayout, id);
                //GetLayoutItem sometimes returns null object
                if (l === undefined || l === null){
                    l = {x:0, y:0}
                }

                if (eventName === "dragmove" || eventName === "dragstart") {
                    this.placeholder.i = id;
                    this.placeholder.x = l.x;
                    this.placeholder.y = l.y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    this.$nextTick(function() {
                        this.isDragging = true;
                    });
                    //this.$broadcast("updateWidth", this.width);
                    this.eventBus.$emit("updateWidth", this.width);
                } else {
                    this.$nextTick(function() {
                        this.isDragging = false;
                    });
                }

                // set localLayout element coordinates to dragged position
                l.x = x;
                l.y = y;
                // Move the element to the dragged location.
                this.localLayout = moveElement(this.localLayout, l, x, y, true);
                compact(this.localLayout, this.verticalCompact);
                // needed because vue can't detect changes on array element properties
                this.eventBus.$emit("compact");
                this.updateHeight();

                if (this.localLayout.length) this.$emit('layout-update', this.localLayout);
                if (eventName === 'dragend') this.$emit('layout-updated', this.localLayout);
            },
            resizeEvent: function (eventName, id, x, y, h, w) {
                if (eventName === "resizestart" || eventName === "resizemove") {
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    this.$nextTick(function() {
                        this.isDragging = true;
                    });
                    //this.$broadcast("updateWidth", this.width);
                    this.eventBus.$emit("updateWidth", this.width);

                } else {
                    this.$nextTick(function() {
                        this.isDragging = false;
                    });
                }
                let l = getLayoutItem(this.localLayout, id);
                //GetLayoutItem sometimes return null object
                if (l === undefined || l === null){
                    l = {h:0, w:0}
                }
                l.h = h;
                l.w = w;

                if (this.responsive) this.responsiveGridLayout();

                compact(this.localLayout, this.verticalCompact);
                this.eventBus.$emit("compact");
                this.updateHeight();

                if (this.localLayout.length) this.$emit('layout-update', this.localLayout);
                if (eventName === 'resizeend') this.$emit('layout-updated', this.localLayout);
            },

            // finds or generates new layouts for set breakpoints
            responsiveGridLayout(){

                let newBreakpoint = getBreakpointFromWidth(this.breakpoints, this.width);
                let newCols = getColsFromBreakpoint(newBreakpoint, this.cols);

                // save actual localLayout in layouts
                if(this.lastBreakpoint != null && !this.layouts[this.lastBreakpoint])
                    this.layouts[this.lastBreakpoint] = cloneLayout(this.localLayout);

                // Find or generate a new localLayout.
                let layout = findOrGenerateResponsiveLayout(
                    this.originalLayout,
                    this.layouts,
                    this.breakpoints,
                    newBreakpoint,
                    this.lastBreakpoint,
                    newCols,
                    this.verticalCompact
                );

                // Store the new layout.
                this.layouts[newBreakpoint] = layout;

                // new prop sync
                this.$emit('update:layout', layout);

                this.lastBreakpoint = newBreakpoint;
                this.eventBus.$emit("setColNum", getColsFromBreakpoint(newBreakpoint, this.cols));
            },

            // clear all responsive layouts
            initResponsiveFeatures(){
                // clear layouts
                this.layouts = {};
            },

            // find difference in layouts
            findDifference(layout, originalLayout){

                //Find values that are in result1 but not in result2
                let uniqueResultOne = layout.filter(function(obj) {
                    return !originalLayout.some(function(obj2) {
                        return obj.i === obj2.i;
                    });
                });

                //Find values that are in result2 but not in result1
                let uniqueResultTwo = originalLayout.filter(function(obj) {
                    return !layout.some(function(obj2) {
                        return obj.i === obj2.i;
                    });
                });

                //Combine the two arrays of unique entries#
                return uniqueResultOne.concat(uniqueResultTwo);
            }
        },
    }
</script>
