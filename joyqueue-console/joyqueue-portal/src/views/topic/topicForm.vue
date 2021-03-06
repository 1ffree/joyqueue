<template>
  <div>
    <d-steps :current='current'>
      <d-step title="步骤1" description="填写主题信息"></d-step>
      <d-step title="步骤2" description="选择Broker"></d-step>
    </d-steps>
    <div class="steps-content" style="margin-top: 15px; border: 1px solid #e9e9e9; border-radius: 6px;background-color:
    #fafafa; text-align: left; padding: 20px 30px 40px 50px; height: 100%">
      <div class="step1" v-show="current===0">
        <div class="stepForm1">
          <d-form ref="form1" :model="formData" :rules="rules.rule1" label-width="110px">
            <d-form-item label="主题英文名：" :error="error.code" prop="code">
              <d-input v-model="formData.code" placeholder="仅支持英文字母大小写、数字、-、_和/" style="width: 70%"></d-input>
            </d-form-item>
            <d-form-item label="命名空间：" prop="namespace">
              <d-select v-model="formData.namespace.code" style="width: 50%">
                <d-option v-for="item in namespaceList" :value="item.code" :key="item.code"></d-option>
              </d-select>
            </d-form-item>
            <d-form-item label="主题类型：" prop="type">
              <d-select v-model="formData.type" :value="0" style="width: 70%" @on-change="handlerTypeChange">
                <d-option :value="0" >普通主题</d-option>
                <d-option :value="1" >广播主题</d-option>
                <d-option :value="2" >顺序主题</d-option>
              </d-select>
            </d-form-item>
            <d-form-item label="分区数量：" prop="partitions">
              <d-input v-model.number="formData.partitions" :disabled="partitionsDisabled" style="width: 70%"></d-input>
            </d-form-item>
            <d-form-item label="选举类型：" prop="electType">
              <d-select v-model.number="formData.electType" style="width: 70%">
                <d-option :value="0" >Raft</d-option>
                <d-option :value="1" >Fix</d-option>
              </d-select>
            </d-form-item>
            <d-form-item label="Broker分组：" prop="brokerGroup">
              <d-select v-model="formData.brokerGroup.id" style="width: 70%" @on-change="handlerBrokerGroupChange">
                <d-option v-for="item in brokerGroupList" :value="item.id" :key="item.id">
                  <span>{{ item.name }}</span>
                  <span style="float:right;color:#ccc">{{ item.code }}</span>
                </d-option>
              </d-select>
            </d-form-item>
            <d-form-item label="申请描述：" prop="description">
              <d-input type="textarea" rows="2" v-model="formData.description"
                           placeholder="请输入申请描述，例如用途等" style="width: 70%"/>
            </d-form-item>
          </d-form>
        </div>
        <div class="step-actions" style="text-align: center">
          <d-button type="primary" @click="next">下一步</d-button>
        </div>
      </div>
      <div class="step2" v-show="current===1">
        <div class="stepForm2">
          <d-form ref="form" :model="formData" :rules="rules.rule2" label-width="100px">
            <add-broker ref="brokers" :model="formData.brokers" @on-choosed-broker="choosedBroker"
                        :urls="addBrokerUrls" :colData="addBrokerColData">
            </add-broker>
          </d-form>
        </div>
        <div class="step-actions" style="text-align: center">
          <d-button type="primary" @click="prev">上一步</d-button>
          <d-button type="primary" @click="confirm()">确定</d-button>
        </div>
      </div>
    </div>
    </div>
  </template>

<script>
import apiRequest from '../../utils/apiRequest.js'
import {deepCopy} from '../../utils/assist.js'
import {getNameRule} from '../../utils/common.js'
import form from '../../mixins/form.js'
import addBroker from './addBroker.vue'

export default {
  name: 'topic-form',
  mixins: [ form ],
  components: {
    addBroker
  },
  props: {
    type: 0, // add or edit form
    data: {
      type: Object,
      default: function () {
        return {
          code: '',
          name: '',
          type: 0,
          namespace: {
            id: '',
            code: ''
          },
          partitions: 24,
          brokerGroup: {
            id: -1,
            code: '',
            name: ''
          },
          electType: 0,
          description: '',
          brokers: []
        }
      }
    },
    addBrokerUrls: {
      type: Object
    },
    addBrokerColData: {
      type: Array
    }
  },
  data () {
    let validateBroker = (rule, value, callback) => {
      if (this.formData.topic.partitions !== undefined && this.formData.topic.brokers !== undefined &&
          this.formData.topic.brokers.length > this.formData.topic.partitions) {
        callback(new Error('勾选的broker数量不能大于分区数量'))
      } else {
        callback()
      }
    }
    let validateBrokerGroup = (rule, value, callback) => {
      if (this.formData.brokerGroup.id < 0) {
        callback(new Error('请选择Broker分组'))
      } else {
        callback()
      }
    }
    return {
      current: 0,
      formData: this.data,
      partitionsDisabled: false,
      namespaceList: [],
      brokerGroupList: [],
      brokerList: [],
      urls: {
        findAllNamespace: `/namespace/findAll`,
        findAllBrokerGroup: `/brokerGroup/findAll`,
        searchBroker: '/broker/search',
        add: '/topic/addWithBrokerGroup'
      },
      rules: {
        rule1: {
          code: [
            {required: true, message: '请输入topic英文名', trigger: 'change'},
            {pattern: /^[a-zA-Z0-9/]+[a-zA-Z0-9/_-]{1,120}[a-zA-Z0-9/]+$/, message: '英文名格式不匹配', trigger: 'change'}
          ],
          name: getNameRule(),
          partitions: [
            {type: 'number', required: true, message: '请输入分区数量', trigger: 'change'}
          ],
          brokerGroup: [
            {validator: validateBrokerGroup, trigger: 'change'}
          ]
        },
        rule2: {
          brokers: [
            {type: 'array', required: true, message: '请至少选择一个Broker', trigger: 'change'},
            {validator: validateBroker, trigger: 'blur'}
          ]
        }
      },
      error: {
        code: ''
      }
    }
  },
  methods: {
    prev () {
      // Jump
      this.current = this.current - 1
    },
    next () {
      // Validate
      this.$refs['form' + (this.current + 1)].validate((valid) => {
        if (valid) {
          // Jump
          this.current = this.current + 1
        } else {
          this.$Message.error('验证不通过，请重新填写！')
        }
      })
    },
    choosedBroker (val) {
      this.formData.brokers = val
    },
    handlerTypeChange (data) {
      if (data === 2) {
        this.partitionsDisabled = true
        this.formData.partitions = 1
      } else {
        this.partitionsDisabled = false
        this.formData.partitions = 5
      }
    },
    handlerBrokerGroupChange (data) {
      this.$refs.brokers.getListByGroup(data)
    },
    getNamespaces () {
      apiRequest.get(this.urls.findAllNamespace).then((data) => {
        this.namespaceList = data.data || []
      })
    },
    getBrokerGroups () {
      apiRequest.get(this.urls.findAllBrokerGroup).then((data) => {
        if (data.data === undefined || data.data.length < 1) {
          this.brokerGroupList = [{id: 0, code: '', name: '全部'}]
        }
        this.brokerGroupList = []
        let allItem = {id: 0, code: '', name: '全部'}
        this.brokerGroupList.push(allItem);
        (data.data || []).forEach(item => {
          this.brokerGroupList.push(item)
        })
        // set default value
        this.formData.brokerGroup.id = this.brokerGroupList[0].id
        this.formData.brokerGroup.code = this.brokerGroupList[0].code
        this.formData.brokerGroup.name = this.brokerGroupList[0].name
        this.handlerBrokerGroupChange(this.formData.brokerGroup.id)
      })
    },
    beforeConfirm () {
      let copyData = deepCopy(this.formData || {})
      return copyData
    }
  },
  computed: {
  },
  mounted () {
    this.getNamespaces()
    this.getBrokerGroups()
  }
}
</script>

<style scoped>

</style>
