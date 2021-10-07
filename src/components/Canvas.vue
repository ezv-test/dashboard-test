<template>
    <div ref="canvas" class="canvas" @click="canvasClick">
        <div class="add-btn">
            <div ref="addTrigger" class="add-trigger" :class="{disabled: draggableProccess}" @mousedown.left="addBlock">
                <span>Listen</span>
            </div>
        </div>
        <Moveable
                ref="scene"
                :style="{width: sceneSize[0] + 'px', height: sceneSize[1] + 'px'}"
                class="scene"
                :class="{_arrowPreview: arrows.proccess}"
                v-bind="scene_config"
                @dragEnd="sceneDragEnd"
                @drag="sceneDrag"
                >
            <Moveable
                    v-for="(block, index) in blocks"
                    class="movable-item"
                    :class="{_overlap: block.overlap, _active: activeBlock_index == index, _arrowStart: arrows.previewLine.from == index}"
                    :ref="'mv'+index"
                    :key="index"
                    v-bind="block_config"
                    @dragStart="blockDragStart($event, index)"
                    @drag="blockDrag"
                    @dragEnd="blockDragEnd">
                <span class="block-id">{{ formatId(index) }}</span>
                <div class="movable-item-inner" @click.stop="blockClick(index)" @mouseover="blockMouseOver(index)" @mouseleave="blockMouseLeave">Listen</div>
            </Moveable>
        </Moveable>
    </div>
</template>

<script>
import Moveable from 'vue-moveable';
import LeaderLine from 'leader-line-new';

LeaderLine.positionByWindowResize = false

export default {
    components: {
        Moveable,
    },
    data: () => ({
        blocks: [], // список основных блоков
        checkTm: null, // таймаут для проверки коллизий при движении блока
        spaceHolder: 30, // место, которое нельзя занимать вокруг блока
        activeBlock_index: 0, // порядковый номер блока, который в данный момент перемещают
        draggableProccess: false,
        draggableEnded: false,
        sceneSize: [5000, 5000], // размер сцены, [x, y]
        block_config: { // конфиг для плагина vue-movable, блоки
            draggable: true,
        },
        scene_config: { // конфиг для плагина vue-movable, сцена
            draggable: true,
            snappable: true,
            bounds: {}
        },
        arrows: {
            proccess: false,
            previewLine: {
                from: null,
                to: null,
                arrow: null
            },
            list: [],
            style: {
                preview: {
                    color: '#781DD9',
                    size: 2,
                    dash: {animation: true}
                },
                stable: {
                    color: '#C0C2C4',
                    size: 2,
                    dash: false
                }
            }
        }
    }),
    computed: {
        activeBlock: {
            set (data) {
                this.$set(this.blocks[this.activeBlock_index], data.key, data.value)
            },
            get () {
                return this.blocks[this.activeBlock_index]
            }
        }
    },
    methods: {
        canvasClick (e) {
            if (e.target.className != 'movable-item-inner') this.cancelPreview()
        },
        formatId (id) {
            return (id + 1).toString().padStart(4, '0')
        },
        /***
         * показываем превью стрелки
         */
        blockMouseOver (index) {
            if (this.arrows.proccess) {
                if (this.arrows.previewLine.from == index || this.hasConnection(this.arrows.previewLine.from, index)) return false

                this.arrows.previewLine.to = index

                this.arrows.previewLine.arrow = this.drawArrow(this.arrows.previewLine.from, this.arrows.previewLine.to, true)
            }
        },
        blockMouseLeave () {
            if (this.arrows.proccess && this.arrows.previewLine.arrow) {
                this.removePreviewLine()
            }
        },
        removePreviewLine () {
            this.arrows.previewLine.arrow.remove()
            this.arrows.previewLine.to = null
            this.arrows.previewLine.arrow = null
        },

        cancelPreview () {
            if (this.arrows.previewLine.arrow) this.arrows.previewLine.arrow.remove()
            this.arrows.previewLine = {
                from: null,
                to: null,
                arrow: null
            }
            this.arrows.proccess = false
        },

        /***
         * проверяем наличие связи между блоками
         */
        hasConnection (from, to) {
            for (let arrow of this.arrows.list) {
                if (arrow.from == from && arrow.to == to) return true
            }
            return false
        },
        /***
         * меняем стили стрелки на базовые серые,
         * переносим на сцену,
         * сохраняем стрелку и добавляем в связанные блоки ее id
         */
        connectBlocks () {
            this.arrows.previewLine.arrow.setOptions(this.arrows.style.stable)

            const lastLine = this.transferLastArrow()

            this.arrows.list.push({
                from: this.arrows.previewLine.from,
                to: this.arrows.previewLine.to,
                DOMelem: lastLine
            })
            let arrow_index = this.arrows.list.length - 1

            this.blocks[this.arrows.previewLine.from].arrows.push(arrow_index)
            this.blocks[this.arrows.previewLine.to].arrows.push(arrow_index)

            this.arrows.previewLine.arrow = null
        },
        /***
         * рисуем сохраненные стрелки при дропе блока
         */
        redrawBlockArrows () {
            for (let arrow_id of this.activeBlock.arrows) {
                let arrow = this.arrows.list[arrow_id]
                this.drawArrow(arrow.from, arrow.to)
                arrow.DOMelem = this.transferLastArrow()
            }
        },
        /***
         * удаляем стрелки при старте драга
         */
        removeBlockArrows () {
            for (let arrow_id of this.activeBlock.arrows) {
                let arrow = this.arrows.list[arrow_id]
                arrow.DOMelem.remove()
                arrow.DOMelem = null
            }
        },
        drawArrow (from, to, preview = false) {
            return new LeaderLine(
                this.blocks[from].movable.$el,
                this.blocks[to].movable.$el,
                preview ? this.arrows.style.preview : this.arrows.style.stable
            )
        },
        /***
         * библиотека позволяет добавлять стрелки только в body,
         * поэтому чтобы не пересчитывать координаты всех стрелок при перемещении сцены,
         * вручную перемещаем каждую стрелку на сцену с пересчетом координат
         */
        transferLastArrow () {
            const lastLine = document.body.querySelector(':scope > .leader-line:last-of-type')
            const line_rect = lastLine.getBoundingClientRect()
            const scene_rect = this.$refs.scene.getRect()

            lastLine.style.left = (line_rect.left - scene_rect.left - 2) + 'px'
            lastLine.style.top = (line_rect.top - scene_rect.top - 2) + 'px'

            this.$refs.scene.$el.appendChild(lastLine);

            return lastLine
        },
        /***
         * нажали на блок и начали построение стрелок
         */
        connectPreviewStart (index) {
            this.arrows.proccess = true
            this.arrows.previewLine.from = index
        },
        /***
         * устанавливаем границы перемещения сцены
         */
        setSceneBounds () {
            this.scene_config.bounds = { left: window.innerWidth - this.sceneSize[0],
                                        right: this.sceneSize[0],
                                        top: window.innerHeight - this.sceneSize[1],
                                        bottom: this.sceneSize[1] + 56}
            this.$nextTick(() => {
                this.$refs.scene.request("draggable", {}, true);
            })
        },
        /***
         * запоминаем активный элемент и задаем начальную позицию при переносе с кнопки
         */
        blockDragStart (e, index) {
            this.activeBlock_index = index
            this.scene_config.draggable = false


            if (typeof this.activeBlock.id == 'undefined') {
                const trigger_top = this.$refs.addTrigger.getBoundingClientRect().top
                const scene_rect = this.$refs.scene.getRect()
                e.setTransform(`translate(${-scene_rect.left}px, ${-scene_rect.top + trigger_top}px)`)
            }
        },
        /***
         * при клике по блоку начинаем рисовать превью стрелки, или окончательно связываем блоки
         */
        blockClick (index) {
            if (this.blocks.length <= 1) return false

            if (this.arrows.previewLine.from == index || this.draggableEnded) {
                this.draggableEnded = false
                this.cancelPreview()
                return false
            }

            if (this.arrows.proccess) {
                if (this.arrows.previewLine.to !== null) {
                    this.connectBlocks()
                }
                this.cancelPreview()
            } else {
                this.connectPreviewStart(index)
            }
        },
        /***
         * при дропе сохраняем, если нет пересечений,
         * удаляем, если это новый элемент,
         * или откатываем на старое место
         */
        blockDragEnd ({ currentTarget }) {
            if (this.draggableProccess) { // элемент протащили
                this.draggableProccess = false

                let isNewBlock = typeof this.activeBlock.id == 'undefined'

                const target = currentTarget.target
                const bounds = target.getBoundingClientRect()

                if (this.checkTm) clearTimeout(this.checkTm)
                if (!this.checkCollision(bounds)) {
                    this.activeBlock = {key: 'id', value: this.activeBlock_index}
                    this.activeBlock = {key: 'prevTransform', value: target.style.transform}
                } else {
                    if (isNewBlock) {
                        this.blocks.splice(this.activeBlock_index, 1)
                    } else {
                        target.style.transform = this.activeBlock.prevTransform;
                    }
                }

                if (!isNewBlock) {
                    this.draggableEnded = true
                    this.redrawBlockArrows()
                }
            }

            this.scene_config.draggable = true
            this.activeBlock_index = null
        },
        /***
         * при перемешении блока удаляем стрелки и проверяем на коллизии
         */
        blockDrag ({ target, transform }) {
            target.style.transform = transform

            if (!this.draggableProccess) { // первый сдвиг
                if (this.arrows.proccess) this.cancelPreview()
                this.removeBlockArrows()
            }
            this.draggableProccess = true

            if (this.checkTm) clearTimeout(this.checkTm)
            this.checkTm = setTimeout(() => {
                this.checkCollision(target.getBoundingClientRect(), true)
            }, 50)
        },

        sceneDrag ({ target, transform }) {
            target.style.transform = transform;
        },
        /***
         * обновляем позиции блоков при перемещении сцены
         */
        sceneDragEnd () {
            for (let block of this.blocks) {
                block.movable.updateRect()
            }
        },

        /***
         * проверяем пересечение активного обьекта с остальными, highlight - выделение цветом при пересечении
         */
        checkCollision (bounds, highlight = false) {
            let hasCollision = false
            for (let block of this.blocks) {
                const blockRect = block.movable.$el.getBoundingClientRect()

                if (typeof block.id == 'undefined' || block.id == this.activeBlock_index) continue

                block.overlap = false
                if (!(blockRect.right + this.spaceHolder < bounds.left ||
                      blockRect.left > bounds.right + this.spaceHolder ||
                      blockRect.bottom + this.spaceHolder < bounds.top ||
                      blockRect.top > bounds.bottom + this.spaceHolder))
                {
                    hasCollision = true
                    if (highlight) block.overlap = true
                }
            }
            return hasCollision
        },
        /***
         * добавляем кнопку и начинаем перетаскивание
         */
        addBlock (e) {
            this.blocks.push({})

            this.$nextTick(() => {
                const last_index = this.blocks.length - 1
                const movable_index = `mv${last_index}`

                this.activeBlock_index = last_index

                this.activeBlock = {key: 'movable', value: this.$refs[movable_index][0]}
                this.activeBlock = {key: 'overlap', value: false}
                this.activeBlock = {key: 'arrows', value: []}

                this.activeBlock.movable.dragStart(e)
            })
        }
    },
    mounted () {
        this.setSceneBounds()
        window.addEventListener("resize", this.setSceneBounds);
    },
    beforeDestroy() {
        window.removeEventListener('resize', this.setSceneBounds);
    },
}
</script>

<style lang="scss">
.add-btn {
    position: absolute;
    top: 24px;
    left: 16px;
    z-index: 100;
    background:white;
    padding:8px;
    box-shadow: 0 1px 1px rgba(0, 0, 0, 0.06), -1px 2px 2px rgba(0, 0, 0, 0.05), -2px 4px 4px rgba(0, 0, 0, 0.04), -4px 8px 8px rgba(0, 0, 0, 0.03), -6px 12px 12px rgba(0, 0, 0, 0.02);
    border-radius: 10px;
    user-select:none;

    .add-trigger {
        padding:14px 11px 14px 9px;
        box-shadow: 0 1px 1px rgba(0, 0, 0, 0.05), 0 1px 2px rgba(0, 0, 0, 0.05);
        border-radius: 8px;
        position:relative;
        cursor:url(~@/assets/images/cursor.svg), auto;
        transition:0.3s;

        &:before, &:after {
            content:'';
            position:absolute;
            right:11px;
            top:50%;
            transform:translate(0,-50%);
            width:2px;
            height:12px;
            background: #C0C2C4;
            border-radius: 1px;
        }
        &:before {
            right:15px;
        }

        span {
            font-size:15px;
            position:relative;
            padding:0 17px 0 31px;
            &:before {
                content:'';
                position:absolute;
                left:0;
                box-sizing:border-box;
                width:22px;
                height:18px;
                top:50%;
                transform:translate(0,-50%);
                background: #F5D9FF;
                border: 2px solid #A353E0;
                border-radius: 2px;
            }
        }
        &.disabled {
            &:before, &:after {
                display:none;
            }
            background:#F4F5F6;
            box-shadow:none;
            span {
                color:#C0C2C4;
                &:before {
                    opacity:0.3;
                }
            }
        }
    }
}
.movable-item {
    width:292px;
    height:100px;
    position:absolute;
    box-sizing:border-box;
    border: 2px solid #781DD9;
    border-radius: 2px;
    background: #FDF5FF;
    user-select:none;
    z-index: 1;
    transition:box-shadow 0.2s, background 0.2s;
    cursor: url(~@/assets/images/cursor.svg), auto;

    div {
        display:flex;
        justify-content:center;
        align-items:center;
        font-size:14px;
        height: 100%;
    }
    span {
        position:absolute;
        top:-27px;
        left:0;
        color:#6B6E71;
        font-size:14px;
        font-family: 'Fira Mono', monospace;
    }
    &._arrowStart {
        box-shadow: 0 10px 12px rgba(81, 85, 90, 0.04), 0 6px 6px rgba(81, 85, 90, 0.04), 0 3px 3px rgba(81, 85, 90, 0.16);
    }
    &._active {
        z-index:10;
    }
    &._overlap {
        background:#ffbaea;
    }
}
.canvas {
    position:relative;
    height:calc(100% - 56px);
    width:100%;
    overflow:hidden;
    display: flex;
    align-items: center;
    justify-content: center;
}
.scene {
    background:url(~@/assets/images/pattern.jpg) center center repeat;
    background-size: auto 16px;
    cursor:move;
    position:absolute;
    border: 2px solid red;
    box-sizing: border-box;
    &._arrowPreview {
        .movable-item {
            cursor:pointer;
        }
    }
}

.moveable-line {
    display: none!important;
}
</style>
