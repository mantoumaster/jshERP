<template>
  <j-modal
    :title="title"
    :width="width"
    :visible="visible"
    :confirmLoading="confirmLoading"
    :keyboard="false"
    :forceRender="true"
    v-bind:prefixNo="prefixNo"
    fullscreen
    switchHelp
    switchFullscreen
    @cancel="handleCancel"
    :id="prefixNo"
    style="top:20px;height: 95%;">
    <template slot="footer">
      <a-button @click="handleCancel">取消</a-button>
      <a-button v-if="billPrintFlag && isShowPrintBtn" @click="handlePrintPro('销售出库')">三联打印-新版</a-button>
      <a-button v-if="billPrintFlag && isShowPrintBtn" @click="handlePrint('销售出库')">三联打印</a-button>
      <a-button v-if="checkFlag && isCanCheck" :loading="confirmLoading" @click="handleOkAndCheck">保存并审核</a-button>
      <a-button type="primary" :loading="confirmLoading" @click="handleOk">保存（Ctrl+S）</a-button>
      <!--发起多级审核-->
      <a-button v-if="!checkFlag" @click="handleWorkflow()" type="primary">提交流程</a-button>
    </template>
    <a-spin :spinning="confirmLoading">
      <a-form :form="form">
        <a-row class="form-row" :gutter="24">
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="客户" data-step="1" data-title="客户"
                         data-intro="客户必须选择，如果发现需要选择的客户尚未录入，可以在下拉框中点击新增客户进行录入。
                          特别注意，客户如果录入之后在下拉框中不显示，请检查是否给当前用户分配对应的客户权限">
              <a-select placeholder="请选择客户" v-decorator="[ 'organId', validatorRules.organId ]" :disabled="!rowCanEdit"
                :dropdownMatchSelectWidth="false" showSearch optionFilterProp="children" @change="handleOrganChange">
                <div slot="dropdownRender" slot-scope="menu">
                  <v-nodes :vnodes="menu" />
                  <a-divider style="margin: 4px 0;" />
                  <div v-if="quickBtn.customer" class="dropdown-btn" @mousedown="e => e.preventDefault()" @click="addCustomer"><a-icon type="plus" /> 新增客户</div>
                  <div class="dropdown-btn" @mousedown="e => e.preventDefault()" @click="initCustomer(0)"><a-icon type="reload" /> 刷新列表</div>
                </div>
                <a-select-option v-for="(item,index) in cusList" :key="index" :value="item.id">
                  {{ item.supplier }}
                </a-select-option>
              </a-select>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="单据日期">
              <j-date v-decorator="['operTime', validatorRules.operTime]" :show-time="true"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="单据编号" data-step="2" data-title="单据编号"
                         data-intro="单据编号自动生成、自动累加、开头是单据类型的首字母缩写，累加的规则是每次打开页面会自动占用一个新的编号">
              <a-input placeholder="请输入单据编号" v-decorator.trim="[ 'number', validatorRules.number ]" />
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="关联订单" data-step="3" data-title="关联订单"
              data-intro="销售出库单据可以通过关联订单来选择已录入的订单，选择之后会自动加载订单的内容，然后继续录入仓库等信息完成单据的提交，
              提交之后原来的销售订单会对应的改变单据状态。另外本系统支持订单多次出库，只需选择订单之后修改对应的商品数量即可">
              <a-input-search placeholder="请选择关联订单" v-decorator="[ 'linkNumber' ]" @search="onSearchLinkNumber" :readOnly="true"/>
            </a-form-item>
          </a-col>
        </a-row>
        <j-editable-table id="billModal"
          :ref="refKeys[0]"
          :loading="materialTable.loading"
          :columns="materialTable.columns"
          :dataSource="materialTable.dataSource"
          :minWidth="minWidth"
          :maxHeight="300"
          :rowNumber="false"
          :rowSelection="true"
          :actionButton="rowCanEdit"
          :actionDeleteButton="!rowCanEdit"
          :dragSortAndNumber="rowCanEdit"
          @valueChange="onValueChange"
          @added="onAdded"
          @deleted="onDeleted">
          <template #buttonAfter>
            <a-row v-if="rowCanEdit" :gutter="24" style="float:left;padding-bottom:5px;padding-right:8px" data-step="4" data-title="扫码录入" data-intro="此功能支持扫码枪扫描商品条码进行录入">
              <a-col v-if="scanStatus" :md="6" :sm="24">
                <a-button @click="scanEnter">扫码录入</a-button>
              </a-col>
              <a-col v-if="!scanStatus" :md="16" :sm="24" style="padding: 0 8px 0 12px">
                <a-input placeholder="请扫条码或序列号并回车" v-model="scanBarCode" @pressEnter="scanPressEnter" ref="scanBarCode"/>
              </a-col>
              <a-col v-if="!scanStatus" :md="6" :sm="24" style="padding: 0px 12px 0 0">
                <a-button @click="stopScan">收起扫码</a-button>
              </a-col>
            </a-row>
            <a-row :gutter="24" style="float:left;padding-bottom: 5px;">
              <a-col :md="24" :sm="24">
                <a-button @click="handleHistoryBillList"><a-icon type="history" />历史单据</a-button>
              </a-col>
            </a-row>
            <a-row v-if="rowCanEdit" :gutter="24" style="float:left;padding-bottom: 5px;padding-left:20px;">
              <a-button icon="import" @click="onImport(prefixNo)">导入明细</a-button>
            </a-row>
          </template>
          <template #depotBatchSet>
            <a-icon type="down" @click="handleBatchSetDepot" />
          </template>
          <template #depotAdd>
            <a-divider v-if="quickBtn.depot" style="margin: 4px 0;" />
            <div v-if="quickBtn.depot" class="dropdown-btn" @click="addDepot"><a-icon type="plus" /> 新增</div>
            <div class="dropdown-btn" @mousedown="e => e.preventDefault()" @click="initDepot"><a-icon type="reload" /> 刷新</div>
          </template>
        </j-editable-table>
        <a-row class="form-row" :gutter="24">
          <a-col :lg="24" :md="24" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="{xs: { span: 24 },sm: { span: 24 }}" label="">
              <a-textarea :rows="1" placeholder="请输入备注" v-decorator="[ 'remark' ]" style="margin-top:8px;"/>
            </a-form-item>
          </a-col>
        </a-row>
        <a-row class="form-row" :gutter="24">
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="优惠率" data-step="5" data-title="优惠率"
                         data-intro="针对单据明细中商品总金额进行优惠的比例">
              <a-input style="width:80%;" placeholder="请输入优惠率" v-decorator.trim="[ 'discount' ]" suffix="%" @change="onChangeDiscount"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="收款优惠" data-step="6" data-title="收款优惠"
                         data-intro="针对单据明细中商品总金额进行优惠的金额">
              <a-input placeholder="请输入付款优惠" v-decorator.trim="[ 'discountMoney' ]" @change="onChangeDiscountMoney"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="优惠后金额" data-step="7" data-title="优惠后金额"
                         data-intro="针对单据明细中商品总金额进行优惠后的金额">
              <a-input placeholder="请输入优惠后金额" v-decorator.trim="[ 'discountLastMoney' ]" :readOnly="true"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="其它费用" data-step="8" data-title="其它费用"
                         data-intro="比如快递费、油费、过路费">
              <a-input placeholder="请输入其它费用" v-decorator.trim="[ 'otherMoney' ]" @change="onChangeOtherMoney"/>
            </a-form-item>
          </a-col>
        </a-row>
        <a-row class="form-row" :gutter="24">
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="结算账户" data-step="9" data-title="结算账户"
                         data-intro="如果在下拉框中选择多账户，则可以通过多个结算账户进行结算">
              <a-select style="width:80%;" placeholder="请选择结算账户" v-decorator="[ 'accountId', validatorRules.accountId ]"
                        :dropdownMatchSelectWidth="false" allowClear @select="selectAccount">
                <div slot="dropdownRender" slot-scope="menu">
                  <v-nodes :vnodes="menu" />
                  <a-divider style="margin: 4px 0;" />
                  <div v-if="quickBtn.account" class="dropdown-btn" @mousedown="e => e.preventDefault()" @click="addAccount"><a-icon type="plus" /> 新增</div>
                  <div class="dropdown-btn" @mousedown="e => e.preventDefault()" @click="initAccount(0)"><a-icon type="reload" /> 刷新</div>
                </div>
                <a-select-option v-for="(item,index) in accountList" :key="index" :value="item.id">
                  {{ item.name }}
                </a-select-option>
              </a-select>
              <a-tooltip title="多账户明细">
                <a-button type="default" icon="folder" style="margin-left: 8px;" size="small" v-show="manyAccountBtnStatus" @click="handleManyAccount"/>
              </a-tooltip>
            </a-form-item>
          </a-col>
          <a-col v-if="depositStatus" :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="扣除订金">
              <a-input v-decorator.trim="[ 'deposit' ]" @change="onChangeDeposit"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="本次收款">
              <a-input placeholder="请输入本次收款" v-decorator.trim="[ 'changeAmount', validatorRules.changeAmount ]" @change="onChangeChangeAmount"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="本次欠款" data-step="10" data-title="本次欠款"
                         data-intro="欠款产生的费用，后续可以在收款单进行收取">
              <a-input placeholder="请输入本次欠款" v-decorator.trim="[ 'debt', validatorRules.price ]" :readOnly="true"/>
            </a-form-item>
          </a-col>
        </a-row>
        <a-row class="form-row" :gutter="24">
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="销售人员" data-step="11" data-title="销售人员"
                         data-intro="销售人员的数据来自【经手人管理】菜单中的销售员">
              <j-select-multiple style="width:80%;" placeholder="请选择销售人员" v-model="personList.value" :options="personList.options"/>
            </a-form-item>
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
          </a-col>
          <a-col :lg="6" :md="12" :sm="24">
          </a-col>
        </a-row>
        <a-row class="form-row" :gutter="24">
          <a-col :lg="6" :md="12" :sm="24">
            <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="附件" data-step="12" data-title="附件"
                         data-intro="可以上传与单据相关的图片、文档，支持多个文件">
              <j-upload v-model="fileList" bizPath="bill"></j-upload>
            </a-form-item>
          </a-col>
        </a-row>
      </a-form>
    </a-spin>
    <many-account-modal ref="manyAccountModalForm" @ok="manyAccountModalFormOk"></many-account-modal>
    <import-item-modal ref="importItemModalForm" @ok="importItemModalFormOk"></import-item-modal>
    <link-bill-list ref="linkBillList" @ok="linkBillListOk"></link-bill-list>
    <customer-modal ref="customerModalForm" @ok="customerModalFormOk"></customer-modal>
    <depot-modal ref="depotModalForm" @ok="depotModalFormOk"></depot-modal>
    <account-modal ref="accountModalForm" @ok="accountModalFormOk"></account-modal>
    <batch-set-depot ref="batchSetDepotModalForm" @ok="batchSetDepotModalFormOk"></batch-set-depot>
    <history-bill-list ref="historyBillListModalForm"></history-bill-list>
    <workflow-iframe ref="modalWorkflow" @ok="workflowModalFormOk"></workflow-iframe>
    <bill-print-iframe ref="modalPrint"></bill-print-iframe>
    <bill-print-pro-iframe ref="modalPrintPro"></bill-print-pro-iframe>
  </j-modal>
</template>
<script>
  import pick from 'lodash.pick'
  import ManyAccountModal from '../dialog/ManyAccountModal'
  import ImportItemModal from '../dialog/ImportItemModal'
  import LinkBillList from '../dialog/LinkBillList'
  import CustomerModal from '../../system/modules/CustomerModal'
  import DepotModal from '../../system/modules/DepotModal'
  import AccountModal from '../../system/modules/AccountModal'
  import BatchSetDepot from '../dialog/BatchSetDepot'
  import HistoryBillList from '../dialog/HistoryBillList'
  import WorkflowIframe from '@/components/tools/WorkflowIframe'
  import BillPrintIframe from '../dialog/BillPrintIframe'
  import BillPrintProIframe from '../dialog/BillPrintProIframe'
  import { FormTypes } from '@/utils/JEditableTableUtil'
  import { JEditableTableMixin } from '@/mixins/JEditableTableMixin'
  import { BillModalMixin } from '../mixins/BillModalMixin'
  import { getMpListShort,handleIntroJs } from "@/utils/util"
  import JSelectMultiple from '@/components/jeecg/JSelectMultiple'
  import JUpload from '@/components/jeecg/JUpload'
  import JDate from '@/components/jeecg/JDate'
  import Vue from 'vue'
  import { getCurrentSystemConfig } from '@/api/api'
  export default {
    name: "SaleOutModal",
    mixins: [JEditableTableMixin, BillModalMixin],
    components: {
      ManyAccountModal,
      ImportItemModal,
      LinkBillList,
      CustomerModal,
      DepotModal,
      AccountModal,
      BatchSetDepot,
      HistoryBillList,
      WorkflowIframe,
      BillPrintIframe,
      BillPrintProIframe,
      JUpload,
      JDate,
      JSelectMultiple,
      VNodes: {
        functional: true,
        render: (h, ctx) => ctx.props.vnodes,
      }
    },
    data () {
      return {
        title:"操作",
        width: '1600px',
        moreStatus: false,
        // 新增时子表默认添加几行空数据
        addDefaultRowNum: 1,
        visible: false,
        operTimeStr: '',
        prefixNo: 'XSCK',
        depositStatus: false,
        fileList:[],
        rowCanEdit: true,
        model: {},
        labelCol: {
          xs: { span: 24 },
          sm: { span: 8 },
        },
        wrapperCol: {
          xs: { span: 24 },
          sm: { span: 16 },
        },
        refKeys: ['materialDataTable', ],
        activeKey: 'materialDataTable',
        materialTable: {
          loading: false,
          dataSource: [],
          columns: [
            { title: '仓库名称', key: 'depotId', width: '8%', type: FormTypes.select, placeholder: '请选择${title}', options: [],
              allowSearch:true, validateRules: [{ required: true, message: '${title}不能为空' }]
            },
            { title: '条码', key: 'barCode', width: '12%', type: FormTypes.popupJsh, kind: 'material', multi: true,
              validateRules: [{ required: true, message: '${title}不能为空' }]
            },
            { title: '名称', key: 'name', width: '10%', type: FormTypes.normal },
            { title: '规格', key: 'standard', width: '9%', type: FormTypes.normal },
            { title: '型号', key: 'model', width: '9%', type: FormTypes.normal },
            { title: '颜色', key: 'color', width: '5%', type: FormTypes.normal },
            { title: '品牌', key: 'brand', width: '6%', type: FormTypes.normal },
            { title: '制造商', key: 'mfrs', width: '6%', type: FormTypes.normal },
            { title: '扩展1', key: 'otherField1', width: '4%', type: FormTypes.normal },
            { title: '扩展2', key: 'otherField2', width: '4%', type: FormTypes.normal },
            { title: '扩展3', key: 'otherField3', width: '4%', type: FormTypes.normal },
            { title: '库存', key: 'stock', width: '5%', type: FormTypes.normal },
            { title: '单位', key: 'unit', width: '4%', type: FormTypes.normal },
            { title: '序列号', key: 'snList', width: '12%', type: FormTypes.popupJsh, kind: 'sn', multi: true },
            { title: '批号', key: 'batchNumber', width: '7%', type: FormTypes.popupJsh, kind: 'batch', multi: false },
            { title: '有效期', key: 'expirationDate',width: '7%', type: FormTypes.input, readonly: true },
            { title: '多属性', key: 'sku', width: '9%', type: FormTypes.normal },
            { title: '原数量', key: 'preNumber', width: '4%', type: FormTypes.normal },
            { title: '已出库', key: 'finishNumber', width: '4%', type: FormTypes.normal },
            { title: '数量', key: 'operNumber', width: '4%', type: FormTypes.inputNumber, statistics: true,
              validateRules: [{ required: true, message: '${title}不能为空' }]
            },
            { title: '单价', key: 'unitPrice', width: '4%', type: FormTypes.inputNumber},
            { title: '金额', key: 'allPrice', width: '5%', type: FormTypes.inputNumber, statistics: true },
            { title: '税率', key: 'taxRate', width: '4%', type: FormTypes.inputNumber,placeholder: '%'},
            { title: '税额', key: 'taxMoney', width: '5%', type: FormTypes.inputNumber, readonly: true, statistics: true },
            { title: '价税合计', key: 'taxLastMoney', width: '7%', type: FormTypes.inputNumber, statistics: true },
            { title: '备注', key: 'remark', width: '6%', type: FormTypes.input },
            { title: '关联id', key: 'linkId', width: '5%', type: FormTypes.hidden },
          ]
        },
        confirmLoading: false,
        validatorRules:{
          operTime:{
            rules: [
              { required: true, message: '请输入单据日期！' }
            ]
          },
          number:{
            rules: [
              { required: true, message: '请输入单据编号!' }
            ]
          },
          organId:{
            rules: [
              { required: true, message: '请选择客户！' }
            ]
          },
          accountId:{
            rules: [
              { required: true, message: '请选择结算账户！' }
            ]
          },
          changeAmount:{
            rules: [
              { required: true, message: '请输入金额，如果为空请填0！' },
              { pattern: /^(([0-9][0-9]*)|([0]\.\d{0,4}|[0-9][0-9]*\.\d{0,4}))$/, message: '金额格式不正确!' }
            ]
          }
        },
        url: {
          add: '/depotHead/addDepotHeadAndDetail',
          edit: '/depotHead/updateDepotHeadAndDetail',
          detailList: '/depotItem/getDetailList'
        }
      }
    },
    created () {
    },
    methods: {
      //调用完edit()方法之后会自动调用此方法
      editAfter() {
        this.billStatus = '0'
        this.currentSelectDepotId = ''
        this.rowCanEdit = true
        this.materialTable.columns[1].type = FormTypes.popupJsh
        this.changeColumnHide()
        this.changeFormTypes(this.materialTable.columns, 'snList', 0)
        this.changeFormTypes(this.materialTable.columns, 'batchNumber', 0)
        this.changeFormTypes(this.materialTable.columns, 'expirationDate', 0)
        this.changeFormTypes(this.materialTable.columns, 'preNumber', 0)
        this.changeFormTypes(this.materialTable.columns, 'finishNumber', 0)
        if (this.action === 'add') {
          this.depositStatus = false
          this.addInit(this.prefixNo)
          this.personList.value = ''
          this.fileList = []
          this.$nextTick(() => {
            handleIntroJs(this.prefixNo, 1)
            if(this.transferParam && this.transferParam.number) {
              let tp = this.transferParam
              this.linkBillListOk(tp.list, tp.number, tp.organId, tp.discountMoney, tp.deposit, tp.remark, this.defaultDepotId, tp.accountId, tp.salesMan)
            }
          })
        } else {
          if(this.model.linkNumber) {
            this.rowCanEdit = false
            this.materialTable.columns[1].type = FormTypes.normal
          }
          this.model.operTime = this.model.operTimeStr
          if(this.model.deposit) {
            this.depositStatus = true
          } else {
            this.depositStatus = false
            this.model.deposit = 0
          }
          this.model.debt = (this.model.discountLastMoney + this.model.otherMoney - this.model.deposit - this.model.changeAmount).toFixed(2)
          if(this.model.accountId == null) {
            this.model.accountId = 0
            this.manyAccountBtnStatus = true
            this.accountIdList = this.model.accountIdList
            this.accountMoneyList = this.model.accountMoneyList
          } else {
            this.manyAccountBtnStatus = false
          }
          this.personList.value = this.model.salesMan
          this.fileList = this.model.fileName
          this.$nextTick(() => {
            this.form.setFieldsValue(pick(this.model,'organId', 'operTime', 'number', 'linkNumber', 'remark',
              'discount','discountMoney','discountLastMoney','otherMoney','accountId','deposit','changeAmount','debt','salesMan'))
          });
          // 加载子表数据
          let params = {
            headerId: this.model.id,
            mpList: getMpListShort(Vue.ls.get('materialPropertyList')),  //扩展属性
            linkType: 'basic'
          }
          let url = this.readOnly ? this.url.detailList : this.url.detailList;
          this.requestSubTableData(url, params, this.materialTable);
        }
        //复制新增单据-初始化单号和日期
        if(this.action === 'copyAdd') {
          this.model.id = ''
          this.model.tenantId = ''
          this.copyAddInit(this.prefixNo)
        }
        this.initSystemConfig()
        this.initCustomer(0)
        this.initSalesman()
        this.initDepot()
        this.initAccount(0)
        this.initPlatform()
        this.initQuickBtn()
        this.handleChangeOtherField()
      },
      //提交单据时整理成formData
      classifyIntoFormData(allValues) {
        let totalPrice = 0
        let billMain = Object.assign(this.model, allValues.formValue)
        let detailArr = allValues.tablesValue[0].values
        billMain.type = '出库'
        billMain.subType = '销售'
        for(let item of detailArr){
          totalPrice += item.allPrice-0
        }
        billMain.totalPrice = totalPrice
        if(billMain.accountId === 0) {
          billMain.accountId = ''
        }
        billMain.accountIdList = this.accountIdList.length>0 ? JSON.stringify(this.accountIdList) : ""
        billMain.accountMoneyList = this.accountMoneyList.length>0 ? JSON.stringify(this.accountMoneyList) : ""
        if(this.fileList && this.fileList.length > 0) {
          billMain.fileName = this.fileList
        } else {
          billMain.fileName = ''
        }
        if(this.model.id){
          billMain.id = this.model.id
        }
        billMain.salesMan = this.personList.value
        billMain.status = this.billStatus
        return {
          info: JSON.stringify(billMain),
          rows: JSON.stringify(detailArr),
        }
      },
      handleHistoryBillList() {
        let organId = this.form.getFieldValue('organId')
        this.$refs.historyBillListModalForm.show('出库', '销售', '客户', organId);
        this.$refs.historyBillListModalForm.disableSubmit = false;
      },
      onSearchLinkNumber() {
        this.$refs.linkBillList.show('其它', '销售订单', '客户', "1,3")
        this.$refs.linkBillList.title = "请选择销售订单"
      },
      linkBillListOk(selectBillDetailRows, linkNumber, organId, discountMoney, deposit, remark, depotId, accountId, salesMan) {
        let that = this
        this.rowCanEdit = false
        this.materialTable.columns[1].type = FormTypes.normal
        this.changeFormTypes(this.materialTable.columns, 'preNumber', 1)
        this.changeFormTypes(this.materialTable.columns, 'finishNumber', 1)
        if(selectBillDetailRows && selectBillDetailRows.length>0) {
          let listEx = []
          let allTaxLastMoney = 0
          for(let j=0; j<selectBillDetailRows.length; j++) {
            let info = selectBillDetailRows[j];
            if(info.finishNumber>0) {
              info.operNumber = info.preNumber - info.finishNumber
              info.allPrice = info.operNumber * info.unitPrice-0
              let taxRate = info.taxRate-0
              info.taxMoney = (info.allPrice*taxRate/100).toFixed(2)-0
              info.taxLastMoney = (info.allPrice + info.taxMoney).toFixed(2)-0
            }
            info.linkId = info.id
            allTaxLastMoney += info.taxLastMoney
            if(info.operNumber>0) {
              listEx.push(info)
              this.changeColumnShow(info)
            }
          }
          this.materialTable.dataSource = listEx
          ///给优惠后金额重新赋值
          allTaxLastMoney = allTaxLastMoney?allTaxLastMoney:0
          let discount = 0
          if(allTaxLastMoney!==0) {
            discount = (discountMoney/allTaxLastMoney*100).toFixed(2)-0
          }
          let discountLastMoney = (allTaxLastMoney - discountMoney).toFixed(2)-0
          let changeAmount = discountLastMoney
          if(deposit) {
            this.depositStatus = true
            changeAmount = (discountLastMoney - deposit).toFixed(2)-0
          }
          this.$nextTick(() => {
            this.form.setFieldsValue({
              'organId': organId,
              'linkNumber': linkNumber,
              'discount': discount,
              'discountMoney': discountMoney,
              'discountLastMoney': discountLastMoney,
              'deposit': deposit,
              'changeAmount': changeAmount,
              'accountId': accountId,
              'remark': remark
            })
            getCurrentSystemConfig().then((res) => {
              if (res.code === 200 && res.data) {
                let flag = res.data.zeroChangeAmountFlag==='1'?true:false
                if(flag) {
                  //切换收付款的金额为0
                  let oldChangeAmount = this.form.getFieldValue('changeAmount')-0
                  this.form.setFieldsValue({'changeAmount':0, 'debt':oldChangeAmount})
                }
              }
            })
            this.personList.value = salesMan
          })
          //判断后进行仓库的切换
          if(depotId) {
            setTimeout(function () {
              that.batchSetDepotModalFormOk(depotId)
            },1000)
          }
        }
      },
    }
  }
</script>
<style scoped>

</style>