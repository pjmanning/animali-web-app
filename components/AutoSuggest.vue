<template>
    <div class="vx-auto-suggest">
        <div class="relative flex items-center h-full">
            <!-- Input -->
            <input
                ref="input"
                :placeholder="placeholder"
                :class="inputClasses"
                class="z-30 h-full"
                v-model="searchQuery"
                @keyup.esc="escPressed"
                @keyup.up="increaseIndex(false)"
                @keyup.down="increaseIndex"
                @keyup.enter="suggestionSelected"
                @focus="updateInputFocus"
                @blur="updateInputFocus(false)"
            />
        </div>

        <!-- Group List -->
        <ul
            ref="scrollContainer"
            :class="{ hidden: !inputFocused }"
            class="z-50 mt-2 overflow-x-hidden rounded-lg shadow-lg auto-suggest-suggestions-list"
            @mouseenter="insideSuggestions = true"
            @mouseleave="insideSuggestions = false"
            @focus="updateInputFocus"
            @blur="updateInputFocus(false)"
            tabindex="-1"
        >
            <li
                ref="grp_list"
                v-for="(suggestion_list, grp_name, grp_index) in filteredData"
                :key="grp_index"
                class="auto-suggest__suggestion-group-container"
            >
                <!-- Group Header -->
                <p
                    class="px-4 pt-3 pb-1 auto-suggest__suggestion-group-title"
                    v-if="!hideGroupTitle"
                >
                    <slot name="group" :group_name="grp_name"></slot>
                </p>

                <!-- Suggestion List of each group -->
                <ul>
                    <li
                        v-for="(suggestion, index) in suggestion_list"
                        :key="index"
                        class="px-4 py-3 cursor-pointer auto-suggest__suggestion-group__suggestion"
                        :class="{
                            'vx-auto-suggest__current-selected':
                                currentSelected == `${grp_index}.${index}`
                        }"
                        @mouseenter="currentSelected = `${grp_index}.${index}`"
                        @click="suggestionSelected"
                    >
                        <slot :name="grp_name" :suggestion="suggestion"></slot>
                    </li>

                    <li
                        class="px-4 py-3 auto-suggest__suggestion-group__suggestion no-results"
                        v-if="!suggestion_list.length && searchQuery"
                    >
                        <slot name="noResult" :group_name="grp_name">
                            <p>No Results Found.</p>
                        </slot>
                    </li>
                </ul>
            </li>
        </ul>
    </div>
</template>

<script>
export default {
    props: {
        placeholder: {
            type: String,
            default: 'Search..'
        },
        data: {
            type: Object,
            required: true
        },
        initalData: {
            type: Object,
            default: () => new Object()
        },
        inputClasses: {
            type: [String, Object, Array]
        },
        autoFocus: {
            type: Boolean,
            default: false
        },
        showPinned: {
            type: Boolean,
            default: false
        },
        searchLimit: {
            type: Number,
            default: 4
        },
        hideGroupTitle: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            searchQuery: '',
            filteredData: {},
            currentSelected: -1,
            inputFocused: false,
            insideSuggestions: false
        }
    },
    watch: {
        // UPDATE SUGGESTIONS LIST
        searchQuery(val) {
            this.$emit('input', val)

            if (val === '') {
                this.inputInit()
            } else {
                const queried_data = {}
                const data_grps = Object.keys(this.data)

                data_grps.forEach((grp, i) => {
                    queried_data[data_grps[i]] = this.filter_grp(this.data[grp])
                })

                // Check if any of group has at least one queried item
                if (!Object.values(queried_data).some(obj => obj.length)) {
                    this.currentSelected = -1
                }

                this.filteredData = queried_data
            }
        },
        autoFocus(val) {
            if (val) this.focusInput()
            else this.searchQuery = ''
        },
        filteredData(val) {
            // Auto Select first item if it's not item-404
            let grp_index = null

            for (const [index, grp_suggestions] of Object.values(
                val
            ).entries()) {
                if (grp_suggestions.length) {
                    grp_index = index
                    break
                }
            }

            if (grp_index !== null) this.currentSelected = `${grp_index}.0`
        }
    },
    methods: {
        escPressed() {
            this.$emit('closeSearchbar')
            this.searchQuery = ''
        },
        filter_grp(grp) {
            // const exactEle = grp.data.filter(item => {
            //     return item[grp.key]
            //         .toLowerCase()
            //         .startsWith(this.searchQuery.toLowerCase())
            // })
            // const containEle = grp.data.filter(item => {
            //     return (
            //         !item[grp.key]
            //             .toLowerCase()
            //             .startsWith(this.searchQuery.toLowerCase()) &&
            //         item[grp.key]
            //             .toLowerCase()
            //             .indexOf(this.searchQuery.toLowerCase()) > -1
            //     )
            // })
            // return exactEle.concat(containEle).slice(0, this.searchLimit)
            return grp.data.filter(animal => {
                animal.searchString = animal.name + ' ' + animal.id
                // eslint-disable-next-line
                return (
                    animal.searchString
                        .toLowerCase()
                        .indexOf(this.searchQuery.toLowerCase()) > -1
                )
            })
        },
        inputInit() {
            if (
                Object.entries(this.initalData).length === 0 &&
                this.initalData.constructor === Object
            ) {
                this.filteredData = {}
            } else {
                this.filteredData = this.initalData
            }
        },
        updateInputFocus(val = true) {
            if (val) {
                if (this.searchQuery === '') this.inputInit()
                setTimeout(() => {
                    this.inputFocused = true
                }, 100)
            } else {
                if (this.insideSuggestions) return
                setTimeout(() => {
                    this.inputFocused = false
                }, 100)
                this.escPressed()
            }
        },
        suggestionSelected() {
            if (this.currentSelected > -1) {
                const [grp_index, item_index] = this.currentSelected.split('.')

                const grp_of_selected_item = Object.keys(this.data)[grp_index]
                const selected_item = this.filteredData[grp_of_selected_item][
                    item_index
                ]

                this.$emit('selected', {
                    [grp_of_selected_item]: selected_item
                })

                this.searchQuery = ''
            }
        },
        increaseIndex(val = true) {
            /* eslint-disable no-lonely-if */

            // If there's no matching items
            if (
                !Object.values(this.filteredData).some(
                    grp_items => grp_items.length
                )
            )
                return

            const [grp_i, item_i] = this.currentSelected.split('.')

            const grp_arr = Object.entries(this.filteredData)
            const active_grp_total_items = grp_arr[grp_i][1].length

            if (val) {
                // If active item is not of last item in grp
                if (active_grp_total_items - 1 > item_i) {
                    this.currentSelected = `${grp_i}.${Number(item_i) + 1}`

                    // If active item grp is not last in grp list
                } else if (grp_i < grp_arr.length - 1) {
                    for (let i = Number(grp_i) + 1; i < grp_arr.length; i++) {
                        // If navigating group have items => Then move in that group
                        if (grp_arr[i][1].length > 0) {
                            this.currentSelected = `${Number(i)}.0`
                            break
                        }
                    }
                }
            } else {
                // If active item is not of first item in grp
                if (Number(item_i)) {
                    this.currentSelected = `${grp_i}.${Number(item_i) - 1}`

                    // If active item grp  is not first in grp list
                } else if (Number(grp_i)) {
                    for (let i = Number(grp_i) - 1; i >= 0; i--) {
                        // If navigating group have items => Then move in that group
                        if (grp_arr[i][1].length > 0) {
                            this.currentSelected = `${i}.${grp_arr[i][1]
                                .length - 1}`
                            break
                        }
                    }
                }
            }
            /* eslint-enable no-lonely-if */
        },
        focusInput() {
            this.$refs.input.$el.querySelector('input').focus()
        }
    },
    mounted() {
        if (this.autoFocus) this.focusInput()
    }
}
</script>

<style>
.vx-auto-suggest {
    position: relative;
}
.vx-auto-suggest .vs-input .vs-con-input .vs-inputx {
    z-index: 10;
}
.vx-auto-suggest .vs-input .vs-con-input .vs-input--placeholder {
    top: 0;
    z-index: 50;
}
.vx-auto-suggest .auto-suggest-suggestions-list {
    position: absolute;
    background: #fff;
    width: 100vw;
    margin-top: 0;
    left: -100px;
    border-radius: 0;
    max-height: 82vh;
    overflow-y: auto;
}

#add-encounter-suggest .auto-suggest-suggestions-list {
    left: 0;
    width: calc(100vw - 2rem);
}

@screen sm {
    .vx-auto-suggest .auto-suggest-suggestions-list {
        width: 100%;
        left: 0;
        @apply rounded-lg;
        @apply mt-1;
    }

    #add-encounter-suggest .auto-suggest-suggestions-list {
        left: 0;
        width: 100%;
    }
}
.auto-suggest__suggestion-group-title {
    @apply bg-gray-50;
}

.vx-auto-suggest
    .auto-suggest-suggestions-list
    .auto-suggest__suggestion-group__suggestion.vx-auto-suggest__current-selected {
    background: #f1f1f1;
}
</style>
