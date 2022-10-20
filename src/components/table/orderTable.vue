<template>
      <a-card>
            <order-table-search @search="(exp) => { tblRef.expr = exp; tbl.manual() }" ref="search">
            </order-table-search>
            <c-table :tblRef="tblRef" ref="tbl" :size="props.size">
                  <template #bodyCell="{ column, text, record }">
                        <template v-if="column.dataIndex === 'type'">
                              <span>{{ text === 0 ? 'DDL' : 'DML' }}</span>
                        </template>
                        <template v-if="column.dataIndex === 'assigned'">
                              <a-tag v-for="i in text.split(',')">{{ i }}</a-tag>
                        </template>
                        <template v-if="column.dataIndex === 'delay'">{{
                                    text === 'none' ? $t('order.table.delay') : text
                        }}</template>
                        <template v-if="column.dataIndex === 'status'">
                              <state-tags :state="text"></state-tags>
                        </template>
                        <template v-if="column.dataIndex === 'action'">
                              <a-button type="primary" size="small" @click="profie(record)">{{ $t('common.profile') }}
                              </a-button>
                              <a-button style="margin-left: 10px;" type="primary" size="small" @click="showModal(record)">
                                To new order
                              </a-button>
                        </template>
                  </template>
            </c-table>
            <a-modal v-model:visible="visible" :footer="null" title="Chose create by env">
                <!-- <a-list :data-source="sourceList">
                    <template #renderItem="{ item }">
                        <a-list-item>
                            <p>{{ item.source }}</p>
                        </a-list-item>
                    </template>
                </a-list> -->
                <!-- <a-table :columns="createByOrderCol" :data-source="sourceList" :pagination="{ hideOnSinglePage : true }" size="middle" /> -->
                <a-table :columns="createByOrderCol" :data-source="sourceList" :pagination="{ hideOnSinglePage : true }" size="middle">
                    <template #bodyCell="{ column, record }">
                        <template v-if="column.title === 'action'">
                            <a-button type="primary" size="small" @click="startByOldOrder(record)">Create it</a-button>
                        </template>
                    </template>
                </a-table>
            </a-modal>
      </a-card>
</template>

<script lang="ts" setup>
import StateTags from "./stateTags.vue"
import OrderTableSearch from "./orderTableSearch.vue"
import { Res } from "@/config/request";
import { onBeforeRouteUpdate, useRoute, useRouter } from "vue-router"
import { AxiosResponse } from "axios";
import { onMounted, reactive, ref } from "vue"
import { Request, OrderTableResp, OrderExpr, OrderParams, } from "@/apis/orderPostApis"
import { OrderTableData } from '@/types'
import { useStore } from '@/store'
import { useI18n } from 'vue-i18n';
import { tableRef } from ".";
import { Request as RequestToSchema } from "@/apis/fetchSchema";
import { RespFetchSource } from "@/apis/listAppApis"

interface propsAttr {
      size?: string
}

const props = withDefaults(defineProps<propsAttr>(), {
      size: "default"
})

const search = ref()

const { t } = useI18n()

const tblRef = reactive<tableRef>({
      col: [
            {
                  title: t('common.table.work_id'),
                  dataIndex: 'work_id',
                  width: 200
            },
            {
                  title: t('common.table.remark'),
                  dataIndex: 'text',
                  ellipsis: true
            },
            {
                  title: t('common.table.type'),
                  dataIndex: 'type',
            },
            {
                  title: t('common.table.post.time'),
                  dataIndex: 'date',
            },
            {
                  title: t('common.table.post.user'),
                  dataIndex: 'username',
            },
            {
                  title: t('common.table.post.real_name'),
                  dataIndex: 'real_name',

            },
            {
                  title: t('order.profile.timing'),
                  dataIndex: 'delay',
            },
            {
                  title: t('order.profile.auditor'),
                  dataIndex: 'assigned',
            },
            {
                  title: t('common.table.state'),
                  dataIndex: 'status',

            },
            {
                  title: t('common.action'),
                  dataIndex: 'action',
                  width: 200,
            }
      ],
      data: [] as OrderTableData[],
      pageCount: 0,
      defaultPageSize: 20,
      expr: {
            status: 7,
            type: 2,
            text: "",
            username: ""
      } as OrderExpr,
      isloop: true,
      fn: (expr: OrderParams) => {
            request.List(expr, isAudit.value).then((res: AxiosResponse<Res<OrderTableResp>>) => {
                  tblRef.data = res.data.payload.data
                  tblRef.pageCount = res.data.payload.page
            })
      }
})

const route = useRoute()

const router = useRouter()

const store = useStore()

const request = new Request

const requestToSchema = new RequestToSchema

const tbl = ref()

const isAudit = ref("")

const createByOrderCol = [
    { title: 'source', dataIndex : 'source' },
    { title: 'idc', dataIndex : 'idc' },
    { title: 'id', dataIndex : 'source_id' },
    { title: 'action' },
]


const visible = ref<boolean>(false)
let sourceList = ref<any[]>(new Array());

const showModal = (record: OrderTableData) => {
    visible.value = true;
    let reqType:string = '';
    switch (record.type) {
    case 0:
        reqType = 'ddl'
        break
    case 1:
        reqType = 'dml'
        break
    }
    requestToSchema.Source(reqType).then((res: AxiosResponse<Res<any[]>>) => {
            sourceList.value = res.data.payload;
            console.log(sourceList)
        }).finally(() => {
            // loading.value = false
        })
    store.commit("order/ORDER_STORE", record);
}

const startByOldOrder = (item : any) => {
      router.push({ 
        path: "/apply/order/old",
        query: { work_id: store.state.order.order.work_id, type: store.state.order.order.type, idc: item.idc, source: item.source, source_id: item.source_id }
     })
}

const profie = (record: OrderTableData) => {
      store.commit("order/ORDER_STORE", record)
      if (route.params.tp === "audit") {
            router.push({ path: "/server/order/audit/profile" })
      } else if (route.params.tp === "common") {
            router.push({ path: "/server/order/common/profile" })
      } else {
            router.push({ path: "/server/order/record/profile" })
      }
}

onBeforeRouteUpdate((to) => {
      isAudit.value = to.params.tp as string
      tbl.value.manual()
})


onMounted(() => {
      isAudit.value = route.params.tp as string
})

</script>