<!-- by j i s h e n g h u a -->
<template>
  <a-row :gutter="24">
    <a-col :md="24">
      <a-card :style="cardStyle" :bordered="false">
        <!-- 查询区域 -->
        <div class="table-page-search-wrapper">
          <!-- 搜索区域 -->
          <a-form layout="inline" @keyup.enter.native="searchQuery">
            <a-row :gutter="24">
              <a-col :md="6" :sm="24">
                <a-form-item label="单据编号" :labelCol="labelCol" :wrapperCol="wrapperCol">
                  <a-input placeholder="请输入单据编号" v-model="queryParam.billNo"></a-input>
                </a-form-item>
              </a-col>
              <a-col :md="6" :sm="24">
                <a-form-item label="单据日期" :labelCol="labelCol" :wrapperCol="wrapperCol">
                  <a-range-picker
                    style="width:100%"
                    v-model="queryParam.createTimeRange"
                    format="YYYY-MM-DD"
                    :placeholder="['开始时间', '结束时间']"
                    @change="onDateChange"
                    @ok="onDateOk"
                  />
                </a-form-item>
              </a-col>
              <a-col :md="6" :sm="24">
                <a-form-item label="付款会员" :labelCol="labelCol" :wrapperCol="wrapperCol">
                  <a-select placeholder="请选择付款会员" showSearch allow-clear optionFilterProp="children" v-model="queryParam.organId">
                    <a-select-option v-for="(item,index) in retailList" :key="index" :value="item.id">
                      {{ item.supplier }}
                    </a-select-option>
                  </a-select>
                </a-form-item>
              </a-col>
              <span style="float: left;overflow: hidden;" class="table-page-search-submitButtons">
                <a-col :md="6" :sm="24">
                  <a-button type="primary" @click="searchQuery">查询</a-button>
                  <a-button style="margin-left: 8px" @click="searchReset">重置</a-button>
                  <a @click="handleToggleSearch" style="margin-left: 8px">
                    {{ toggleSearchStatus ? '收起' : '展开' }}
                    <a-icon :type="toggleSearchStatus ? 'up' : 'down'"/>
                  </a>
                </a-col>
              </span>
            </a-row>
            <template v-if="toggleSearchStatus">
              <a-row :gutter="24">
                <a-col :md="6" :sm="24">
                  <a-form-item label="操作员" :labelCol="labelCol" :wrapperCol="wrapperCol">
                    <a-select placeholder="请选择操作员" showSearch allow-clear optionFilterProp="children" v-model="queryParam.creator">
                      <a-select-option v-for="(item,index) in userList" :key="index" :value="item.id">
                        {{ item.userName }}
                      </a-select-option>
                    </a-select>
                  </a-form-item>
                </a-col>
                <a-col :md="6" :sm="24">
                  <a-form-item label="财务人员" :labelCol="labelCol" :wrapperCol="wrapperCol">
                    <a-select placeholder="请选择财务人员" showSearch allow-clear optionFilterProp="children" v-model="queryParam.handsPersonId">
                      <a-select-option v-for="(item,index) in personList" :key="index" :value="item.id">
                        {{ item.name }}
                      </a-select-option>
                    </a-select>
                  </a-form-item>
                </a-col>
                <a-col :md="6" :sm="24">
                  <a-form-item label="单据状态" :labelCol="labelCol" :wrapperCol="wrapperCol">
                    <a-select placeholder="请选择单据状态" allow-clear v-model="queryParam.status">
                      <a-select-option value="0">未审核</a-select-option>
                      <a-select-option value="9" v-if="!checkFlag">审核中</a-select-option>
                      <a-select-option value="1">已审核</a-select-option>
                    </a-select>
                  </a-form-item>
                </a-col>
                <a-col :md="6" :sm="24">
                  <a-form-item label="单据备注" :labelCol="labelCol" :wrapperCol="wrapperCol">
                    <a-input placeholder="请输入单据备注" v-model="queryParam.remark"></a-input>
                  </a-form-item>
                </a-col>
              </a-row>
            </template>
          </a-form>
        </div>
        <!-- 操作按钮区域 -->
        <div class="table-operator"  style="margin-top: 5px">
          <a-button v-if="btnEnableList.indexOf(1)>-1" @click="myHandleAdd" type="primary" icon="plus">新增</a-button>
          <a-button v-if="btnEnableList.indexOf(1)>-1" icon="delete" @click="batchDel">删除</a-button>
          <a-button v-if="checkFlag && btnEnableList.indexOf(2)>-1" icon="check" @click="batchSetStatus(1)">审核</a-button>
          <a-button v-if="checkFlag && btnEnableList.indexOf(7)>-1" icon="stop" @click="batchSetStatus(0)">反审核</a-button>
          <a-button v-if="isShowExcel && btnEnableList.indexOf(3)>-1" icon="download" @click="handleExport">导出</a-button>
          <a-tooltip placement="left" title="针对会员模块，对会员收取预付款。" slot="action">
            <a-icon v-if="btnEnableList.indexOf(1)>-1" type="question-circle" style="font-size:20px;float:right;" />
          </a-tooltip>
        </div>
        <!-- table区域-begin -->
        <div>
          <a-table
            ref="table"
            size="middle"
            bordered
            rowKey="id"
            :columns="columns"
            :dataSource="dataSource"
            :components="handleDrag(columns)"
            :pagination="ipagination"
            :scroll="scroll"
            :loading="loading"
            :rowSelection="{selectedRowKeys: selectedRowKeys, onChange: onSelectChange}"
            @change="handleTableChange">
            <span slot="action" slot-scope="text, record">
              <a @click="myHandleDetail(record, '收预付款', prefixNo)">查看</a>
              <a-divider v-if="btnEnableList.indexOf(1)>-1" type="vertical" />
              <a v-if="btnEnableList.indexOf(1)>-1" @click="myHandleEdit(record)">编辑</a>
              <a-divider v-if="btnEnableList.indexOf(1)>-1" type="vertical" />
              <a-popconfirm v-if="btnEnableList.indexOf(1)>-1" title="确定删除吗?" @confirm="() => myHandleDelete(record)">
                <a>删除</a>
              </a-popconfirm>
            </span>
            <template slot="customRenderStatus" slot-scope="status">
              <a-tag v-if="status == '0'" color="red">未审核</a-tag>
              <a-tag v-if="status == '1'" color="green">已审核</a-tag>
              <a-tag v-if="status == '9'" color="orange">审核中</a-tag>
            </template>
          </a-table>
        </div>
        <!-- table区域-end -->
        <!-- 表单区域 -->
        <advance-in-modal ref="modalForm" @ok="modalFormOk" @close="modalFormClose"></advance-in-modal>
        <financial-detail ref="modalDetail" @ok="modalFormOk" @close="modalFormClose"></financial-detail>
        <bill-excel-iframe ref="billExcelIframe" @ok="modalFormOk" @close="modalFormClose"></bill-excel-iframe>
      </a-card>
    </a-col>
  </a-row>
</template>
<script>
  import AdvanceInModal from './modules/AdvanceInModal'
  import FinancialDetail from './dialog/FinancialDetail'
  import BillExcelIframe from '@/components/tools/BillExcelIframe'
  import { JeecgListMixin } from '@/mixins/JeecgListMixin'
  import { FinancialListMixin } from './mixins/FinancialListMixin'
  import JDate from '@/components/jeecg/JDate'
  import Vue from 'vue'
  export default {
    name: "AdvanceInList",
    mixins:[JeecgListMixin, FinancialListMixin],
    components: {
      AdvanceInModal,
      FinancialDetail,
      BillExcelIframe,
      JDate
    },
    data () {
      return {
        labelCol: {
          span: 5
        },
        wrapperCol: {
          span: 18,
          offset: 1
        },
        // 查询条件
        queryParam: {
          billNo: "",
          searchMaterial: "",
          type: "收预付款",
          organId: undefined,
          creator: undefined,
          handsPersonId: undefined,
          status: undefined,
          remark: ""
        },
        prefixNo: 'SYF',
        // 表头
        columns: [
          {
            title: '操作',
            dataIndex: 'action',
            width:200,
            align:"center",
            scopedSlots: { customRender: 'action' },
          },
          { title: '付款会员', dataIndex: 'organName',width:140, ellipsis:true},
          { title: '单据编号', dataIndex: 'billNo',width:160},
          { title: '单据日期 ', dataIndex: 'billTimeStr',width:160},
          { title: '操作员', dataIndex: 'userName',width:100, ellipsis:true},
          { title: '财务人员', dataIndex: 'handsPersonName',width:100},
          { title: '合计金额', dataIndex: 'totalPrice',width:80},
          { title: '收款金额', dataIndex: 'changeAmount',width:80},
          { title: '备注', dataIndex: 'remark',width:200},
          { title: '状态', dataIndex: 'status', width: 80, align: "center",
            scopedSlots: { customRender: 'customRenderStatus' }
          }
        ],
        url: {
          list: "/accountHead/list",
          delete: "/accountHead/delete",
          deleteBatch: "/accountHead/deleteBatch",
          batchSetStatusUrl: "/accountHead/batchSetStatus"
        }
      }
    },
    computed: {
    },
    created () {
      this.initSystemConfig()
      this.initRetail()
      this.initUser()
      this.initPerson()
      this.initAccount()
    },
    methods: {
    }
  }
</script>
<style scoped>
  @import '~@assets/less/common.less'
</style>