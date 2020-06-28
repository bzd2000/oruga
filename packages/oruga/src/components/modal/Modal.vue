<template>
    <transition
        :name="animation"
        @after-enter="afterEnter"
        @before-leave="beforeLeave"
        @after-leave="afterLeave"
    >
        <div
            v-if="!destroyed"
            v-show="isActive"
            :class="rootClasses"
            v-trap-focus="trapFocus"
            tabindex="-1"
            :role="ariaRole"
            :aria-modal="ariaModal">
            <div :class="backgroundClass" @click="cancel('outside')"/>
            <div
                :class="[ !custom && contentClass ]"
                :style="customStyle">
                <component
                    v-if="component"
                    v-bind="props"
                    v-on="events"
                    :is="component"
                    @close="close"/>
                <div v-else-if="content"> {{ content }} </div>
                <slot v-else/>
                <button
                    type="button"
                    v-if="showX"
                    v-show="!animating"
                    :class="closeClass"
                    @click="cancel('x')"/>
            </div>
        </div>
    </transition>
</template>

<script>
import trapFocus from '../../directives/trapFocus'
import { removeElement, getValueByPath, getCssClass } from '../../utils/helpers'
import config from '../../utils/config'

/**
 * Classic modal overlay to include any content you may need
 * @displayName Modal
 * @style _modal.scss
 */
export default {
    name: 'OModal',
    directives: {
        trapFocus
    },
    props: {
        active: Boolean,
        component: [Object, Function],
        content: String,
        programmatic: Boolean,
        props: Object,
        events: Object,
        width: {
            type: [String, Number],
            default: 960
        },
        custom: Boolean,
        animation: {
            type: String,
            default: 'zoom-out'
        },
        canCancel: {
            type: [Array, Boolean],
            default: () => {
                getValueByPath(config, 'modal.canCancel', ['escape', 'x', 'outside', 'button'])
            }
        },
        onCancel: {
            type: Function,
            default: () => {}
        },
        scroll: {
            type: String,
            default: () => {
                return getValueByPath(config, 'modal.scroll', 'keep')
            },
            validator: (value) => {
                return [ 'clip', 'keep' ].indexOf(value) >= 0
            }
        },
        fullScreen: Boolean,
        trapFocus: {
            type: Boolean,
            default: () => {
                return getValueByPath(config, 'modal.trapFocus', true)
            }
        },
        ariaRole: {
            type: String,
            validator: (value) => {
                return [ 'dialog', 'alertdialog' ].indexOf(value) >= 0
            }
        },
        ariaModal: Boolean,
        destroyOnHide: {
            type: Boolean,
            default: true
        },
        rootClass: {
            type: String,
            default: () => {
                const override = getValueByPath(config, 'modal.override', false)
                const clazz = getValueByPath(config, 'modal.rootClass', '')
                return getCssClass(clazz, override, 'o-modal')
            }
        },
        backgroundClass: {
            type: String,
            default: () => {
                const override = getValueByPath(config, 'modal.override', false)
                const clazz = getValueByPath(config, 'modal.backgroundClas', '')
                return getCssClass(clazz, override, 'o-modal-background')
            }
        },
        contentClass: {
            type: String,
            default: () => {
                const override = getValueByPath(config, 'modal.override', false)
                const clazz = getValueByPath(config, 'modal.contentClass', '')
                return getCssClass(clazz, override, 'o-modal-content')
            }
        },
        closeClass: {
            type: String,
            default: () => {
                const override = getValueByPath(config, 'modal.override', false)
                const clazz = getValueByPath(config, 'modal.closeClass', '')
                return getCssClass(clazz, override, 'o-modal-close')
            }
        },
        fullScreenClass: {
            type: String,
            default: () => {
                const override = getValueByPath(config, 'modal.override', false)
                const clazz = getValueByPath(config, 'modal.fullScreenClass', '')
                return getCssClass(clazz, override, 'o-modal-fullscreen')
            }
        }
    },
    data() {
        return {
            isActive: this.active || false,
            savedScrollTop: null,
            newWidth: typeof this.width === 'number'
                ? this.width + 'px'
                : this.width,
            animating: true,
            destroyed: !this.active
        }
    },
    computed: {
        rootClasses() {
            return [
                this.rootClass,
                this.fullScreen && this.fullScreenClass
            ]
        },
        cancelOptions() {
            return typeof this.canCancel === 'boolean'
                ? this.canCancel
                    ? getValueByPath(config, 'modal.canCancel', ['escape', 'x', 'outside', 'button'])
                    : []
                : this.canCancel
        },
        showX() {
            return this.cancelOptions.indexOf('x') >= 0
        },
        customStyle() {
            if (!this.fullScreen) {
                return { maxWidth: this.newWidth }
            }
            return null
        }
    },
    watch: {
        active(value) {
            this.isActive = value
        },
        isActive(value) {
            if (value) this.destroyed = false
            this.handleScroll()
            this.$nextTick(() => {
                if (value && this.$el && this.$el.focus) {
                    this.$el.focus()
                }
            })
        }
    },
    methods: {
        handleScroll() {
            if (typeof window === 'undefined') return

            if (this.scroll === 'clip') {
                if (this.isActive) {
                    document.documentElement.classList.add('is-clipped')
                } else {
                    document.documentElement.classList.remove('is-clipped')
                }
                return
            }

            this.savedScrollTop = !this.savedScrollTop
                ? document.documentElement.scrollTop
                : this.savedScrollTop

            if (this.isActive) {
                document.body.classList.add('is-noscroll')
            } else {
                document.body.classList.remove('is-noscroll')
            }

            if (this.isActive) {
                document.body.style.top = `-${this.savedScrollTop}px`
                return
            }

            document.documentElement.scrollTop = this.savedScrollTop
            document.body.style.top = null
            this.savedScrollTop = null
        },

        /**
        * Close the Modal if canCancel and call the onCancel prop (function).
        */
        cancel(method) {
            if (this.cancelOptions.indexOf(method) < 0) return

            this.onCancel.apply(null, arguments)
            this.close()
        },

        /**
        * Call the onCancel prop (function).
        * Emit events, and destroy modal if it's programmatic.
        */
        close() {
            this.$emit('close')
            this.$emit('update:active', false)

            // Timeout for the animation complete before destroying
            if (this.programmatic) {
                this.isActive = false
                setTimeout(() => {
                    this.$destroy()
                    removeElement(this.$el)
                }, 150)
            }
        },

        /**
        * Keypress event that is bound to the document.
        */
        keyPress({ key }) {
            if (this.isActive && (key === 'Escape' || key === 'Esc')) this.cancel('escape')
        },

        /**
        * Transition after-enter hook
        */
        afterEnter() {
            this.animating = false
        },

        /**
        * Transition before-leave hook
        */
        beforeLeave() {
            this.animating = true
        },

        /**
        * Transition after-leave hook
        */
        afterLeave() {
            if (this.destroyOnHide) {
                this.destroyed = true
            }
        }
    },
    created() {
        if (typeof window !== 'undefined') {
            document.addEventListener('keyup', this.keyPress)
        }
    },
    beforeMount() {
        // Insert the Modal component in body tag
        // only if it's programmatic
        this.programmatic && document.body.appendChild(this.$el)
    },
    mounted() {
        if (this.programmatic) this.isActive = true
        else if (this.isActive) this.handleScroll()
    },
    beforeDestroy() {
        if (typeof window !== 'undefined') {
            document.removeEventListener('keyup', this.keyPress)
            // reset scroll
            document.documentElement.classList.remove('is-clipped')
            const savedScrollTop = !this.savedScrollTop
                ? document.documentElement.scrollTop
                : this.savedScrollTop
            document.body.classList.remove('is-noscroll')
            document.documentElement.scrollTop = savedScrollTop
            document.body.style.top = null
        }
    }
}
</script>