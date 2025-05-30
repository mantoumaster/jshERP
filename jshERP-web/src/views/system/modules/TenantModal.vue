<template>
  <div ref="container">
    <a-modal
      :title="title"
      :width="600"
      :visible="visible"
      :confirmLoading="confirmLoading"
      :getContainer="() => $refs.container"
      :maskStyle="{'top':'93px','left':'154px'}"
      :wrapClassName="wrapClassNameInfo()"
      :mask="isDesktop()"
      :maskClosable="false"
      @ok="handleOk"
      @cancel="handleCancel"
      cancelText="取消"
      okText="保存"
      style="top:15%;height: 60%;">
      <a-spin :spinning="confirmLoading">
        <a-form :form="form">
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="登录名称">
            <a-input placeholder="请输入登录名称" v-decorator.trim="[ 'loginName', validatorRules.loginName]" :readOnly="!!model.id"
                     suffix="初始密码：123456" />
          </a-form-item>
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="用户数量限制">
            <a-input-number style="width:100%" placeholder="请输入用户数量限制" v-decorator.trim="[ 'userNumLimit' ]" />
          </a-form-item>
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="租户角色" v-if="model.id">
            <a-select style="width:100%" placeholder="请选择租户角色" v-decorator.trim="[ 'roleId' ]">
              <a-select-option v-for="(item,index) in tenantRoleList" :key="index" :value="item.id">
                {{ item.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="租户类型" v-if="model.id">
            <a-select style="width:100%" placeholder="请选择租户类型" v-decorator.trim="[ 'type' ]">
              <a-select-option value="0">试用租户</a-select-option>
              <a-select-option value="1">付费租户</a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="到期时间">
            <j-date style="width:100%" placeholder="请选择到期时间" v-decorator.trim="[ 'expireTime' ]" :show-time="true"/>
          </a-form-item>
          <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="备注">
            <a-textarea :rows="2" placeholder="请输入备注（微信号）" v-decorator.trim="[ 'remark' ]" />
          </a-form-item>
        </a-form>
      </a-spin>
    </a-modal>
  </div>
</template>
<script>
  import pick from 'lodash.pick'
  import {mixinDevice} from '@/utils/mixin'
  import {addTenant,editTenant,checkTenant, getTenantRoleList } from '@/api/api'
  import JDate from '@/components/jeecg/JDate'
  import md5 from 'md5'
  export default {
    name: "TenantModal",
    mixins: [mixinDevice],
    components: {
      JDate
    },
    data () {
      return {
        title:"操作",
        visible: false,
        model: {},
        tenantRoleList: [],  //租户角色列表
        labelCol: {
          xs: { span: 24 },
          sm: { span: 5 },
        },
        wrapperCol: {
          xs: { span: 24 },
          sm: { span: 16 },
        },
        confirmLoading: false,
        form: this.$form.createForm(this),
        validatorRules:{
          loginName:{
            rules: [
              { required: true, message: '请输入登录名称!' },
              { min: 2, max: 30, message: '长度在 2 到 30 个字符', trigger: 'blur' },
              { validator: this.validateLoginName}
            ]
          }
        },
      }
    },
    created () {
    },
    methods: {
      add () {
        this.edit({});
      },
      edit (record) {
        this.form.resetFields();
        this.model = Object.assign({}, record);
        this.model.expireTime = this.model.expireTimeStr
        this.visible = true;
        this.$nextTick(() => {
          this.form.setFieldsValue(pick(this.model,'loginName', 'userNumLimit', 'type', 'roleId', 'expireTime', 'remark'))
        })
        this.getTenantRoleList()
      },
      getTenantRoleList() {
        getTenantRoleList().then((res)=>{
          if(res) {
            this.tenantRoleList = res
          }
        })
      },
      close () {
        this.$emit('close');
        this.visible = false;
      },
      handleOk () {
        const that = this;
        // 触发表单验证
        this.form.validateFields((err, values) => {
          if (!err) {
            that.confirmLoading = true;
            let formData = Object.assign(this.model, values);
            let obj;
            if(!this.model.id){
              formData.password = md5('123456')
              obj=addTenant(formData);
            }else{
              obj=editTenant(formData);
            }
            obj.then((res)=>{
              if(res.code === 200){
                that.$emit('ok');
              }else{
                that.$message.warning(res.data.message);
              }
            }).finally(() => {
              that.confirmLoading = false;
              that.close();
            })
          }
        })
      },
      handleCancel () {
        this.close()
      },
      validateLoginName(rule, value, callback){
        let params = {
          name: value,
          id: this.model.id?this.model.id:0
        };
        checkTenant(params).then((res)=>{
          if(res && res.code===200) {
            if(!res.data.status){
              callback();
            } else {
              callback("登录名称已经存在");
            }
          } else {
            callback(res.data);
          }
        });
      }
    }
  }
</script>
<style scoped>

</style>