<template>
    <section v-if="Object.keys(field_list).length && (!hasInjectComponent || componentsInjected)">

        <slot
            name="beforeFilters"
            :data="data"
            :formData="formData"
            :selectedData="selection?multipleSelection:[]"
        ></slot>

        <filters
            :filters="filters"
            :field_list="field_list"
            :filterOperators="filterOperators"
            @search="getListInfo"
            ref="filters"
        ></filters>

        <slot
            name="afterFilters"
            :data="data"
            :formData="formData"
            :selectedData="selection?multipleSelection:[]"
        ></slot>


        <el-table
            v-if="fields.length && data.length"
            :data="data"
            style="width:100%;"
            @sort-change="handleSortChange"
            @selection-change="handleSelectionChange"
            v-bind="tableConfig"
        >
            <el-table-column
                v-if="selection"
                type="selection"
            ></el-table-column>
            <el-table-column
                v-for="field in fields"
                v-if="field_list[field]"
                :key="field"
                :label="field_list[field]['label']"
                :prop="field"
                :sortable="sortFields.includes(field)?'custom':false"
                v-bind="field_list[field].tableColumnConfig || {}"
            >
                <template slot-scope="scope">
                    
                    <views
                        :descriptor="field_list[field]"
                        :record="scope.row"
                        :field="field"
                    >
                        <template
                            v-if="field_list[field].view && field_list[field].view.component"
                            slot-scope="viewScope"
                        >
                            <component
                                :is="field_list[field].view.name"
                                v-bind="viewScope"
                            ></component>
                        </template>
                    </views>

                </template>
            </el-table-column>
            <el-table-column
                :label="operatorsLabel"
                v-if="operators.length && data.length"
                :min-width="operatorMinWidth"
            >
                <template slot-scope="scope">
                    <operators
                        :field_list="field_list"
                        :operators="operators"
                        :data="scope.row"
                        :index="scope.$index"
                        @update="getListInfo"
                        @setWidth="setOperatorWidth"
                    ></operators>
                </template>
            </el-table-column>
        </el-table>
        <section 
            v-if="data.length === 0"
            style="margin:10px;text-align:center;font-size:28px;color:#777;letter-spacing:8px;"
        >
            {{emptyText}}
        </section>
        <el-pagination
            v-if="paginated && data.length"
            @size-change="handleSizeChange"
            v-bind="paginationConfig"
            :current-page.sync="pageIndex"
            :page-size="localPageSize"
            :total="total"
            style="display:table;margin-left:auto;margin-right:auto;"
        ></el-pagination>
    </section>
</template>

<script>
import filters from "@/components/common/editor/filters"
import operators from "@/components/common/operators/operators"
import views from "@/components/common/views/views"

import mergeAttrsConfig from "@/mixins/common/mergeAttrsConfig.js"
import injectComponents from "@/widget/injectComponents"

import {logError} from "@/widget/utility.js"



export default{
    mixins:[
        mergeAttrsConfig,
    ],
    components:{
        filters,
        operators,
        views,
    },
    beforeCreate(){
        Object.defineProperty(this,'formData',{
            get(){
                return this.$refs.filters && this.$refs.filters.formData;
            }
        });
    },
    created(){
        this.localPageSize = this.pageSize;

        this.needInjectViewComponents = Object.keys(this.field_list).filter((field)=>{
            return this.field_list[field].view && this.field_list[field].view.component;
        }).map(field=>this.field_list[field].view);
        this.hasInjectComponent = this.needInjectViewComponents.length>0;
        this.injectViewComponents();



        this.$watch(()=>{
            return {
                localPageSize:this.localPageSize,
                sortField:this.sortField,
                sortOrder:this.sortOrder,
                pageIndex:this.pageIndex,
            }
        },this.getListInfo)

        this.getListInfo();
    },
    data(){
        return {
            componentsInjected:false,
            data:[],
            pageIndex:1,
            sortField:'',
            sortOrder:'',
            total:0,
            fields:[],
            multipleSelection:[],
            operatorMinWidth:0,
            localPageSize:0,
        }
    },
    props:{
        field_list:{
            type:Object,
            required:true,
        },

        filters:{
            type:Array,
            default:function(){
                return [];
            },
        },
        filterOperators:{
            type:Array,
            default:function(){
                return [];
            }
        },

        pageSize:{
            type:Number,
            default:20
        },
        pageIndexReqName:{
            type:String,
            default:"pageIndex",
        },
        pageSizeReqName:{
            type:String,
            default:"pageSize",
        },
        sortFieldReqName:{
            type:String,
            default:"sortField",
        },
        sortOrderReqName:{
            type:String,
            default:"sortOrder",
        },
        transformRequestData:{
            type:Function,
            default:function(data){
                return data;
            }
        },
        listRequest:{
            type:Function,
            required:true,
        },
        transformListData:{
            type:Function,
            default:async (data)=>{
                return data
            }
        },

        tableConfig:{
            type:Object,
            default(){
                return {};
            }
        },
        selection:{
            type:Boolean,
            default:false,
        },
        sortFields:{
            type:Array,
            default:function(){
                return [];
            }
        },

        operators:{
            type:Array,
            default:function(){
                return [];
            }
        },
        operatorsLabel:{
            type:String,
            default:"操作"
        },

        emptyText:{
            type:String,
            default:"暂无数据"
        },

        paginated:{
            type:Boolean,
            default:true,
        },
        paginationConfig:{
            type:Object,
            default(){
                return {}
            },
        },
    },
    methods:{
        injectViewComponents(){
            if(!this.hasInjectComponent){
                return;
            }

            injectComponents(this,this.needInjectViewComponents).then(()=>{
                this.componentsInjected = true;
            });
        },
        handleSortChange(sortInfo){
            if(sortInfo.prop){
                this.sortField = sortInfo.prop;
                this.sortOrder = sortInfo.order.startsWith('desc')?'desc':'asc';
            }else{
                this.sortField = '';
                this.sortOrder = '';
            }
            this.pageIndex = 1;

        },
        handleSizeChange(newPageSize){
            this.localPageSize = newPageSize;
            this.pageIndex = 1;
        },
        handleSelectionChange(section){
            this.multipleSelection = section;
        },
        getListInfo(){
            let params = {};
            // 有filters 要拿到filters的formData，所以需要等到filters组件实例化完成
            // 有时filters组件虽然实例化了，但是formData数据为空，需要等待formData对应数据形成
            // console.log(this.$refs.filters,!this.$refs.filters || this.$refs.filters.formData )
            if(this.filters.length && ( (!this.$refs.filters) || (Object.keys(this.$refs.filters.formData).length === 0)
             )   ){


                // 这里之所以用setTimeout而不是$nextTick
                // 是因为$nextTick会优先尝试使用Promise
                // 当getListInfo作为microTask的一个任务时
                // 会向同一个microTask push新任务
                // 而挂载子组件到$refs上是作为macroTask的任务
                // 于是会死循环
                setTimeout(()=>{
                    this.getListInfo();
                },0)
                return;

            }

            if(this.filters.length){
                params = Object.assign(params,this.$refs.filters.formData)
            }

            if(this.paginated){
                params[this.pageIndexReqName] = this.pageIndex;
                params[this.pageSizeReqName] = this.localPageSize<=0?
                    this.pageSize:
                    this.localPageSize ;
            }
            
            params[this.sortFieldReqName] = this.sortField;
            params[this.sortOrderReqName] = this.sortOrder;

            console.log(params);

            return new Promise((resolve,reject)=>{
                this.listRequest(resolve,this.transformRequestData(params))
            }).then((rst)=>{
                let {data,total,fields} = rst;
                let promise = this.transformListData(data);
                if(!(promise instanceof Promise)){
                    promise = Promise.resolve(promise);
                }

                promise.then((data)=>{
                    this.fields = fields;
                    this.total = total;
                    this.data = data;
                });

            }).catch(logError);

        },
        setOperatorWidth(width){
            if(width>this.operatorMinWidth){
                this.operatorMinWidth = width;
            }
        },
    },
}
</script>