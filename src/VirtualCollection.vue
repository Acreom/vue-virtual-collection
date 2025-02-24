<style scoped>
.vue-virtual-collection {
    overflow: scroll;
    -webkit-overflow-scrolling: touch;
}

.vue-virtual-collection-container {
    position: relative;
}

.vue-virtual-collection .cell-container {
    position: absolute;
    top: 0;
    left: 0;
}
</style>

<template>
    <div class="vue-virtual-collection" :style="outerStyle" @scroll.passive="onScroll" ref="outer">
        <slot name="header"></slot>
        <div class="vue-virtual-collection-container" :style="containerStyle">
            <div v-for="item in displayItems" class="cell-container" :key="item.key" :style="getComputedStyle(item)">
                <slot name="cell" :data="item.data"></slot>
            </div>
        </div>
    </div>
</template>

<script>
import GroupManager from "./GroupManager"

export default {
    props: {
        cellSizeAndPositionGetter: {
            type: Function,
            required: true
        },
        collection: {
            type: Array,
            required: true
        },
        height: {
            type: Number,
            required: true,
            validator (value) {
                return value >= 0
            }
        },
        width: {
            type: Number,
            required: true,
            validator (value) {
                return value >= 0
            }
        },
        sectionSize: {
            type: Number,
            default: 300
        },
        containerPaddingBottom: {
            type: Number,
            default: 0
        },
        scrollToBottomRange: {
            type: Number
        },
        scrollTopThreshold: {
          type: Number
        },
        headerSlotHeight: {
            type: Number,
            default: 0
        }
    },
    data () {
        return {
            inRangeHit: false,
            totalHeight: 0,
            totalWidth: 0,
            displayItems: []
        }
    },
    watch: {
        collection () {
            this.resetCollection()
        }
    },
    created () {
        this.groupManagers = []
        this.onCollectionChanged()
    },
    methods: {
        resetCollection () {
            // Dispose previous groups and reset associated data
            this.groupManagers.forEach(manager => manager.dispose())
            this.groupManagers = []
            this.totalHeight = 0
            this.totalWidth = 0

            this.onCollectionChanged()
        },
        onCollectionChanged () {
            let collection = this.collection

            // If the collection is flat, wrap it inside a single group
            if (collection.length > 0 && collection[0].group === undefined) {
                collection = [{ group: collection }]
            }

            // Create and store managers for each item group
            collection.forEach((groupContainer, i) => {
                const groupIndex = i // Capture group index for closure
                const unwatch = this.$watch(
                    () => groupContainer,
                    () => this.onGroupChanged(groupContainer.group, groupIndex),
                    { deep: true }
                )

                this.groupManagers.push(new GroupManager(
                    groupContainer.group,
                    groupIndex,
                    this.sectionSize,
                    this.cellSizeAndPositionGetter,
                    unwatch
                ))
            })

            this.updateGridDimensions()
            this.flushDisplayItems()
        },
        updateGridDimensions () {
            this.totalHeight = Math.max(...this.groupManagers.map(it => it.totalHeight))
            this.totalWidth = Math.max(...this.groupManagers.map(it => it.totalWidth))
        },
        onGroupChanged (group, index) {
            this.groupManagers[index].updateGroup(group)
            this.updateGridDimensions()
            this.flushDisplayItems()
        },
        getComputedStyle (displayItem) {
            if (!displayItem) return

            // Currently displayed items may no longer exist
            // if collection has been modified since
            const groupManager = this.groupManagers[displayItem.groupIndex]
            if (!groupManager) return

            const cellMetadatum = groupManager.getCellMetadata(displayItem.itemIndex)
            if (!cellMetadatum) return

            const { width, height, x, y, style } = cellMetadatum
            return {
                transform: `translateX(${x}px) translateY(${y}px)`,
                width: `${width}px`,
                height: `${height}px`,
                ...style
            }
        },
        onScroll (event) {
            this.flushDisplayItems()
            const target = event.target
            const total = target.scrollHeight - target.offsetHeight
            this.$emit("scroll", target)
            if (target.scrollTop === 0) {
            // We have hit the top. Emit event and don't continue.
                this.$emit("scrolled-to-top", target)
                if (this.scrollTopThreshold) return
            }  else if (target.scrollTop === total) {
            //     We have hit the bottom. Emit event and don't continue.
                this.$emit("scrolled-to-bottom", target)
                if (this.scrollToBottomRange) return
            }
            if (this.scrollToBottomRange) {
                // We want to scroll sooner than hitting the bottom
                if (target.scrollTop < total && target.scrollTop > (total - this.scrollToBottomRange)) {
                    // We are in the range
                    if (this.inRangeHit === false) {
                        // If we didn't hit the range previously, emit event
                        this.inRangeHit = true
                        this.$emit("scrolled-to-bottom-range", target)
                    }
                } else {
                    // Reset range hit
                    this.inRangeHit = false
                }
            }
            if (this.scrollTopThreshold) {
              //   We want to scroll sooner than hitting the top
              if (target.scrollTop <= this.scrollTopThreshold) {
                // We are in the range
                  if (this.inRangeHit === false) {
                      // If we didn't hit the range previously, emit event
                      this.inRangeHit = true
                      this.$emit("scrolled-to-top-threshold", target)
                  }
                  else {
                    // Reset range hit
                    this.inRangeHit = false
                  }
              }
            }
        },
        onContainerResized () {
            this.resetCollection()
        },
        flushDisplayItems () {
            let scrollTop = 0
            let scrollLeft = 0
            if (this.$refs.outer) {
                scrollTop = this.$refs.outer.scrollTop - this.headerSlotHeight
                scrollLeft = this.$refs.outer.scrollLeft
            }

            const displayItems = []
            this.groupManagers.forEach((groupManager, groupIndex) => {
                var indices = groupManager.getCellIndices({
                    height: this.height,
                    width: this.width,
                    x: scrollLeft,
                    y: scrollTop
                })

                indices.forEach(itemIndex => {
                    displayItems.push(Object.freeze({
                        groupIndex,
                        itemIndex,
                        key: displayItems.length,
                        ...groupManager.getItem(itemIndex)
                    }))
                })
            })

            if (window.requestAnimationFrame) {
                window.requestAnimationFrame(() => {
                    this.displayItems = displayItems
                    this.$forceUpdate()
                })
            } else {
                this.displayItems = displayItems
                this.$forceUpdate()
            }
        }
    },
    mounted () {
        if (ResizeObserver) {
            this.resizeObserver = new ResizeObserver(this.onContainerResized)
            this.resizeObserver.observe(this.$refs.outer)
        } else {
            this.$refs.outer.addEventListener("resize", this.onContainerResized)
        }
    },
    beforeDestroy () {
        if (ResizeObserver) {
            this.resizeObserver.disconnect()
        } else {
            this.$refs.outer.removeEventListener("resize", this.onContainerResized)
        }
    },
    computed: {
        containerStyle () {
            return {
                height: this.totalHeight + this.containerPaddingBottom + "px",
                width: this.totalWidth + "px"
            }
        },
        outerStyle () {
            return {
                height: this.height + "px",
                width: this.width + "px"
            }
        }
    }
}
</script>
