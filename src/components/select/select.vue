<template>
    <div :class="classes" v-clickoutside="handleClose">
        <div
            :class="[prefixCls + '-selection']"
            v-el:reference
            @click="toggleMenu">
            <div class="ivu-tag" v-for="item in selectedMultiple">
                <span class="ivu-tag-text">{{ item.label }}</span>
                <Icon type="ios-close-empty" @click.stop="removeTag($index)"></Icon>
            </div>
            <span :class="[prefixCls + '-placeholder']" v-show="showPlaceholder && !filterable">{{ placeholder }}</span>
            <span :class="[prefixCls + '-selected-value']" v-show="!showPlaceholder && !multiple && !filterable">{{ selectedSingle }}</span>
            <input
                type="text"
                v-if="filterable"
                v-model="query"
                :class="[prefixCls + '-input']"
                :placeholder="showPlaceholder ? placeholder : ''"
                :style="inputStyle"
                @blur="handleBlur"
                @keydown="resetInputState"
                @keydown.delete="handleInputDelete"
                v-el:input>
            <Icon type="ios-close" :class="[prefixCls + '-arrow']" v-show="showCloseIcon" @click.stop="clearSingleSelect"></Icon>
            <Icon type="arrow-down-b" :class="[prefixCls + '-arrow']"></Icon>
        </div>
        <Dropdown v-show="visible" transition="slide-up" v-ref:dropdown>
            <ul v-show="notFound" :class="[prefixCls + '-not-found']"><li>{{ notFoundText }}</li></ul>
            <ul v-else :class="[prefixCls + '-dropdown-list']" v-el:options><slot></slot></ul>
        </Dropdown>
    </div>
</template>
<script>
    import Icon from '../icon';
    import Dropdown from './dropdown.vue';
    import clickoutside from '../../directives/clickoutside';
    import { oneOf, MutationObserver } from '../../utils/assist';
    import { t } from '../../locale';

    const prefixCls = 'ivu-select';

    export default {
        components: { Icon, Dropdown },
        directives: { clickoutside },
        props: {
            model: {
                type: [String, Number, Array],
                default: ''
            },
            multiple: {
                type: Boolean,
                default: false
            },
            disabled: {
                type: Boolean,
                default: false
            },
            clearable: {
                type: Boolean,
                default: false
            },
            placeholder: {
                type: String,
                default () {
                    return t('i.select.placeholder');
                }
            },
            filterable: {
                type: Boolean,
                default: false
            },
            filterMethod: {
                type: Function
            },
            size: {
                validator (value) {
                    return oneOf(value, ['small', 'large', 'default']);
                }
            },
            labelInValue: {
                type: Boolean,
                default: false
            },
            notFoundText: {
                type: String,
                default () {
                    return t('i.select.noMatch');
                }
            }
        },
        data () {
            return {
                prefixCls: prefixCls,
                visible: false,
                options: [],
                optionInstances: [],
                selectedSingle: '',    // label
                selectedMultiple: [],
                focusIndex: 0,
                query: '',
                inputLength: 20,
                notFound: false,
                slotChangeDuration: false    // if slot change duration and in multiple, set true and after slot change, set false
            };
        },
        computed: {
            classes () {
                return [
                    `${prefixCls}`,
                    {
                        [`${prefixCls}-visible`]: this.visible,
                        [`${prefixCls}-disabled`]: this.disabled,
                        [`${prefixCls}-multiple`]: this.multiple,
                        [`${prefixCls}-single`]: !this.multiple,
                        [`${prefixCls}-show-clear`]: this.showCloseIcon,
                        [`${prefixCls}-${this.size}`]: !!this.size
                    }
                ];
            },
            showPlaceholder () {
                let status = false;

                if ((typeof this.model) === 'string') {
                    if (this.model === '') {
                        status = true;
                    }
                } else if (Array.isArray(this.model)) {
                    if (!this.model.length) {
                        status = true;
                    }
                }

                return status;
            },
            showCloseIcon () {
                return !this.multiple && this.clearable && !this.showPlaceholder;
            },
            inputStyle () {
                let style = {};

                if (this.multiple) {
                    if (this.showPlaceholder) {
                        style.width = '100%';
                    } else {
                        style.width = `${this.inputLength}px`;
                    }
                }

                return style;
            }
        },
        methods: {
            toggleMenu () {
                if (this.disabled) {
                    return false;
                }

                this.visible = !this.visible;
            },
            hideMenu () {
                this.visible = false;
                this.focusIndex = 0;
                this.$broadcast('on-select-close');
            },
            // find option component
            findChild (cb) {
                const find = function (child) {
                    const name = child.$options.componentName;

                    if (name) {
                        cb(child);
                    } else if (child.$children.length) {
                        child.$children.forEach((innerChild) => {
                            find(innerChild, cb);
                        });
                    }
                };

                if (this.optionInstances.length) {
                    this.optionInstances.forEach((child) => {
                        find(child);
                    });
                } else {
                    this.$children.forEach((child) => {
                        find(child);
                    });
                }
            },
            updateOptions (init, slot = false) {
                let options = [];
                let index = 1;

                this.findChild((child) => {
                    options.push({
                        value: child.value,
                        label: (child.label === undefined) ? child.$el.innerHTML : child.label
                    });
                    child.index = index++;

                    if (init) {
                        this.optionInstances.push(child);
                    }
                });

                this.options = options;

                if (init) {
                    this.updateSingleSelected(true, slot);
                    this.updateMultipleSelected(true, slot);
                }
            },
            updateSingleSelected (init = false, slot = false) {
                const type = typeof this.model;

                if (type === 'string' || type === 'number') {
                    let findModel = false;

                    for (let i = 0; i < this.options.length; i++) {
                        if (this.model === this.options[i].value) {
                            this.selectedSingle = this.options[i].label;
                            findModel = true;
                            break;
                        }
                    }

                    if (slot && !findModel) {
                        this.model = '';
                        this.query = '';
                    }
                }

                this.toggleSingleSelected(this.model, init);
            },
            clearSingleSelect () {
                if (this.showCloseIcon) {
                    this.findChild((child) => {
                        child.selected = false;
                    });
                    this.model = '';

                    if (this.filterable) {
                        this.query = '';
                    }
                }
            },
            updateMultipleSelected (init = false, slot = false) {
                if (this.multiple && Array.isArray(this.model)) {
                    let selected = [];

                    for (let i = 0; i < this.model.length; i++) {
                        const model = this.model[i];

                        for (let j = 0; j < this.options.length; j++) {
                            const option = this.options[j];

                            if (model === option.value) {
                                selected.push({
                                    value: option.value,
                                    label: option.label
                                });
                            }
                        }
                    }

                    this.selectedMultiple = selected;

                    if (slot) {
                        let selectedModel = [];

                        for (let i = 0; i < selected.length; i++) {
                            selectedModel.push(selected[i].value);
                        }

                        // if slot change and remove a selected option, emit user
                        if (this.model.length === selectedModel.length) {
                            this.slotChangeDuration = true;
                        }

                        this.model = selectedModel;
                    }
                }
                this.toggleMultipleSelected(this.model, init);
            },
            removeTag (index) {
                if (this.disabled) {
                    return false;
                }
                this.model.splice(index, 1);

                if (this.filterable && this.visible) {
                    this.$els.input.focus();
                }

                this.$broadcast('on-update-popper');
            },
            // to select option for single
            toggleSingleSelected (value, init = false) {
                if (!this.multiple) {
                    let label = '';

                    this.findChild((child) => {
                        if (child.value === value) {
                            child.selected = true;
                            label = (child.label === undefined) ? child.$el.innerHTML : child.label;
                        } else {
                            child.selected = false;
                        }
                    });

                    this.hideMenu();

                    if (!init) {
                        if (this.labelInValue) {
                            this.$emit('on-change', {
                                value: value,
                                label: label
                            });
                            this.$dispatch('on-form-change', {
                                value: value,
                                label: label
                            });
                        } else {
                            this.$emit('on-change', value);
                            this.$dispatch('on-form-change', value);
                        }
                    }
                }
            },
            // to select option for multiple
            toggleMultipleSelected (value, init = false) {
                if (this.multiple) {
                    let hybridValue = [];
                    for (let i = 0; i < value.length; i++) {
                        hybridValue.push({
                            value: value[i]
                        });
                    }

                    this.findChild((child) => {
                        const index = value.indexOf(child.value);

                        if (index >= 0) {
                            child.selected = true;
                            hybridValue[index].label = (child.label === undefined) ? child.$el.innerHTML : child.label;
                        } else {
                            child.selected = false;
                        }
                    });

                    if (!init) {
                        if (this.labelInValue) {
                            this.$emit('on-change', hybridValue);
                            this.$dispatch('on-form-change', hybridValue);
                        } else {
                            this.$emit('on-change', value);
                            this.$dispatch('on-form-change', value);
                        }
                    }
                }
            },
            handleClose () {
                this.hideMenu();
            },
            handleKeydown (e) {
                if (this.visible) {
                    const keyCode = e.keyCode;
                    // Esc slide-up
                    if (keyCode === 27) {
                        e.preventDefault();
                        this.hideMenu();
                    }
                    // next
                    if (keyCode === 40) {
                        e.preventDefault();
                        this.navigateOptions('next');
                    }
                    // prev
                    if (keyCode === 38) {
                        e.preventDefault();
                        this.navigateOptions('prev');
                    }
                    // enter
                    if (keyCode === 13) {
                        e.preventDefault();

                        this.findChild((child) => {
                            if (child.isFocus) {
                                child.select();
                            }
                        });
                    }
                }
            },
            navigateOptions (direction) {
                if (direction === 'next') {
                    const next = this.focusIndex + 1;
                    this.focusIndex = (this.focusIndex === this.options.length) ? 1 : next;
                } else if (direction === 'prev') {
                    const prev = this.focusIndex - 1;
                    this.focusIndex = (this.focusIndex <= 1) ? this.options.length : prev;
                }

                let child_status = {
                    disabled: false,
                    hidden: false
                };

                let find_deep = false;    // can next find allowed

                this.findChild((child) => {
                    if (child.index === this.focusIndex) {
                        child_status.disabled = child.disabled;
                        child_status.hidden = child.hidden;

                        if (!child.disabled && !child.hidden) {
                            child.isFocus = true;
                        }
                    } else {
                        child.isFocus = false;
                    }

                    if (!child.hidden && !child.disabled) {
                        find_deep = true;
                    }
                });

                this.resetScrollTop();

                if ((child_status.disabled || child_status.hidden) && find_deep) {
                    this.navigateOptions(direction);
                }
            },
            resetScrollTop () {
                const index = this.focusIndex - 1;
                let bottomOverflowDistance = this.optionInstances[index].$el.getBoundingClientRect().bottom - this.$refs.dropdown.$el.getBoundingClientRect().bottom;
                let topOverflowDistance = this.optionInstances[index].$el.getBoundingClientRect().top - this.$refs.dropdown.$el.getBoundingClientRect().top;

                if (bottomOverflowDistance > 0) {
                    this.$refs.dropdown.$el.scrollTop += bottomOverflowDistance;
                }
                if (topOverflowDistance < 0) {
                    this.$refs.dropdown.$el.scrollTop += topOverflowDistance;
                }
            },
            handleBlur () {
                setTimeout(() => {
                    const model = this.model;

                    if (this.multiple) {
                        this.query = '';
                    } else {
                        if (model !== '') {
                            this.findChild((child) => {
                                if (child.value === model) {
                                    this.query = child.label === undefined ? child.searchLabel : child.label;
                                }
                            });
                        } else {
                            this.query = '';
                        }
                    }
                }, 300);
            },
            resetInputState () {
                this.inputLength = this.$els.input.value.length * 12 + 20;
            },
            handleInputDelete () {
                if (this.multiple && this.model.length && this.query === '') {
                    this.removeTag(this.model.length - 1);
                }
            },
            // use when slot changed
            slotChange () {
                this.options = [];
                this.optionInstances = [];
            },
            setQuery (query) {
                if (!this.filterable) return;
                this.query = query;
            }
        },
        ready () {
            if (!this.multiple && this.filterable && this.model) {
                this.findChild((child) => {
                    if (this.model === child.value) {
                        if (child.label) {
                            this.query = child.label;
                        } else if (child.searchLabel) {
                            this.query = child.searchLabel;
                        } else {
                            this.query = child.value;
                        }
                    }
                });
            }

            this.updateOptions(true);
            document.addEventListener('keydown', this.handleKeydown);

            // watch slot changed
            if (MutationObserver) {
                this.observer = new MutationObserver(() => {
                    this.slotChange();
                    this.updateOptions(true, true);
                });

                this.observer.observe(this.$els.options, {
//                attributes: true,
                    childList: true,
                    characterData: true,
                    subtree: true
                });
            }
        },
        beforeDestroy () {
            document.removeEventListener('keydown', this.handleKeydown);
            if (this.observer) {
                this.observer.disconnect();
            }
        },
        watch: {
            model () {
                if (this.multiple) {
                    if (this.slotChangeDuration) {
                        this.slotChangeDuration = false;
                    } else {
                        this.updateMultipleSelected();
                    }
                } else {
                    this.updateSingleSelected();
                }
            },
            visible (val) {
                if (val) {
                    if (this.multiple && this.filterable) {
                        this.$els.input.focus();
                    }
                    this.$broadcast('on-update-popper');
                } else {
                    if (this.filterable) {
                        this.$els.input.blur();
                    }
                    this.$broadcast('on-destroy-popper');
                }
            },
            query (val) {
                this.$broadcast('on-query-change', val);
                let is_hidden = true;

                this.$nextTick(() => {
                    this.findChild((child) => {
                        if (!child.hidden) {
                            is_hidden = false;
                        }
                    });
                    this.notFound = is_hidden;
                });
            }
        },
        events: {
            'on-select-selected' (value) {
                if (this.model === value) {
                    this.hideMenu();
                } else {
                    if (this.multiple) {
                        const index = this.model.indexOf(value);
                        if (index >= 0) {
                            this.removeTag(index);
                        } else {
                            this.model.push(value);
                            this.$broadcast('on-update-popper');
                        }

                        if (this.filterable) {
                            this.query = '';
                            this.$els.input.focus();
                        }
                    } else {
                        this.model = value;

                        if (this.filterable) {
                            this.findChild((child) => {
                                if (child.value === value) {
                                    this.query = child.label === undefined ? child.searchLabel : child.label;
                                }
                            });
                        }
                    }
                }
            }
        }
    };
</script>
