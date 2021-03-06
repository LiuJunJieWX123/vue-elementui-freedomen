<template>
    <el-form 
        :label-width="config && config.labelWidth || '120px'" 
        :inline="config && config.inline"
        :size="config && config.size"
        :label-position="config && config.labelPosition"
        class="fd_form">
        <fd-region 
            :columns="tempColumns"
            :data="tempData"
            @event="event"
        >
        </fd-region>
    </el-form>
</template>
<script>
import util from '../../utils/util.js';
import FdRegion from '../../core/region';
import rule from '../../utils/rule.js';

export default {
    name: 'FdForm',
    props: {
        columns: {
            type: Array,
            required: true
        },
        data: {
            type: Object,
            default: function () {
                return {}
            }
        },
        config: Object,
    },
    components: {
        FdRegion
    },
    watch: {
        tempData(nd, od) { 
            if (od) { 
                this.resetColumns(this.columns, nd)
                this.ruleQueues = []
            }
        },
        data(nd, od) { 
            this.tempData = nd
        }
    },
    data() {
        return {
            tempColumns: [],
            tempData: {},
            rules: {}
        }
    },
    methods: {
        resetColumns(columns) {
            let newColumns = []
            columns.forEach(column => {
                if (util.isPlainObject(column)) {
                    let newItem = [column]
                    if (column.rule) {
                        newItem.push( 
                            {type: 'icon', value: 'el-icon-loading', load: () => this.rules[column.prop].loading},
                            {type: 'span', class: 'el-form-item__error', filter: () => this.rules[column.prop].message}  
                        )
                    }
                    newItem.push({type: 'formitem', prop: column.prop, label: column.label, $loadRelation: column})
                    newColumns.push(newItem)
                } else if (Array.isArray(column)) {
                    if (column.length && column[column.length - 1].rule !== void 0) {
                        column.splice(
                            column.length - 1, 0, 
                            // {type: 'icon', value: 'el-icon-loading', load: () => this.rules[column[column.length - 1].prop].loading},
                            {type: 'span', class: 'el-form-item__error', filter: () => this.rules[column[column.length - 1].prop].message}
                        )
                    }
                    newColumns.push(column)
                }
            })
            return newColumns
        },
        resetRules(columns, rules) {
            for (let column of columns) {
                if (Array.isArray(column)) {
                    this.resetRules(column, rules)
                } else if (column.rule !== void 0) {
                    // use index if no prop of form-item?
                    rules[column.prop] = {
                        rule: column.rule,
                        column: column,
                        loading: false,
                        message: ''
                    }
                }
            }
        },
        asyncQueue() {   
            let promises = this.ruleQueues.map(el => {
                return rule.valid(el.data, el.rule, this.data)
            }) 
            return Promise.all(promises)
        },
        validOne(data, r, ruleObj) { 
            if (!r.column.$load) { 
                return false
            }

            let message = rule.valid(data, r.rule, this.data) 
            if (message instanceof Promise) { 
                if (this.ruleQueues) {
                    this.ruleQueues.push({
                        data: data, 
                        rule: r.rule,
                        ruleObj: ruleObj
                    })
                } 
                ruleObj.loading = true 
                message.then(res => {
                    ruleObj.loading = false
                    if (res)
                        ruleObj.message = res 
                })
                return false
            }
            if (message === null || message === void 0 || message === '') {
                return false
            }
            return message
        },
        valid() {
            let error = false
            this.ruleQueues = []
            for (let r in this.rules) {
                let message = this.validOne(this.tempData[r], this.rules[r], this.rules[r])
                if (message !== false) {
                    error = true
                    this.rules[r].message = message
                } else {
                    this.rules[r].message = ''
                }
            }
            return !error
        },
        reset() {
            let newObj = {} 
            for (let key in this.firstData) {
                newObj[key] = this.firstData[key]
            }
            this.tempData = newObj
            for (let r in this.rules) {
                this.rules[r].message = ''
            } 
        },
        clone(columns) {
            return columns
        },
        event(params) { 
            this.$emit('event', params)

            if (params.prop == '$submit') { 
                if (this.valid()) {
                    if (this.ruleQueues.length) {
                        if (!this.submitLoading) {
                            this.submitLoading = true
                        } else {
                            return
                        }
                        this.asyncQueue().then(el => { 
                            if (el.every(el => !el)) 
                                this.$emit('submit', this.tempData)
                            this.submitLoading = false
                        }) 
                    } else {
                        this.$emit('submit', this.tempData)
                    }
                }
            } else if (params.prop == '$reset') {
                this.reset()
            } else if (params.type === 'change' && this.rules[params.prop] !== void 0) {
                let message = this.validOne(this.tempData[params.prop], this.rules[params.prop], this.rules[params.prop])
                if (message) {
                    this.rules[params.prop].message = message
                } else {
                    this.rules[params.prop].message = ''
                }
            } 
        }
    },
    mounted() {
        this.firstData = util.clone(this.data)  
    },
    created() {  
        let columns = this.clone(this.columns)
        let rules = {}
        this.resetRules(columns, rules)
        this.rules = rules  
        this.tempColumns = this.resetColumns(columns)
        this.tempData = this.data
    }
}
</script> 
