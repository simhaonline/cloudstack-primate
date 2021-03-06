// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

<template>
  <a-spin :spinning="loading">
    <a-alert
      v-if="disableSettings"
      banner
      :message="$t('message.action.settings.warning.vm.running')" />
    <div v-else>
      <div v-show="!showAddDetail">
        <a-button
          type="dashed"
          style="width: 100%"
          icon="plus"
          :disabled="!('updateTemplate' in $store.getters.apis && 'updateVirtualMachine' in $store.getters.apis && isAdminOrOwner())"
          @click="showAddDetail = true">
          {{ $t('label.add.setting') }}
        </a-button>
      </div>
      <div v-show="showAddDetail">
        <a-auto-complete
          style="width: 100%"
          :filterOption="filterOption"
          :value="newKey"
          :dataSource="Object.keys(detailOptions)"
          :placeholder="$t('label.name')"
          @change="e => onAddInputChange(e, 'newKey')" />
        <a-auto-complete
          style="width: 100%"
          :filterOption="filterOption"
          :value="newValue"
          :dataSource="detailOptions[newKey]"
          :placeholder="$t('label.value')"
          @change="e => onAddInputChange(e, 'newValue')" />
        <p v-if="error" style="color: red"> {{ $t(error) }} </p>
        <a-button type="primary" style="width: 25%" icon="plus" @click="addDetail">{{ $t('label.add.setting') }}</a-button>
        <a-button type="dashed" style="width: 25%" icon="close" @click="showAddDetail = false">{{ $t('label.cancel') }}</a-button>
      </div>
    </div>
    <a-list size="large">
      <a-list-item :key="index" v-for="(item, index) in details">
        <a-list-item-meta>
          <span slot="title">
            {{ item.name }}
          </span>
          <span slot="description" style="word-break: break-all">
            <span v-if="item.edit" style="display: flex">
              <a-auto-complete
                style="width: 100%"
                :value="item.value"
                :dataSource="detailOptions[item.name]"
                @change="val => handleInputChange(val, index)"
                @pressEnter="e => updateDetail(index)" />
            </span>
            <span v-else>{{ item.value }}</span>
          </span>
        </a-list-item-meta>
        <div slot="actions" v-if="!disableSettings && 'updateTemplate' in $store.getters.apis && 'updateVirtualMachine' in $store.getters.apis && isAdminOrOwner()">
          <a-button shape="circle" size="default" @click="updateDetail(index)" v-if="item.edit">
            <a-icon type="check-circle" theme="twoTone" twoToneColor="#52c41a" />
          </a-button>
          <a-button shape="circle" size="default" @click="hideEditDetail(index)" v-if="item.edit">
            <a-icon type="close-circle" theme="twoTone" twoToneColor="#f5222d" />
          </a-button>
          <a-button
            shape="circle"
            icon="edit"
            v-if="!item.edit"
            @click="showEditDetail(index)" />
        </div>
        <div slot="actions" v-if="!disableSettings && 'updateTemplate' in $store.getters.apis && 'updateVirtualMachine' in $store.getters.apis && isAdminOrOwner()">
          <a-popconfirm
            title="Delete setting?"
            @confirm="deleteDetail(index)"
            okText="Yes"
            cancelText="No"
            placement="left"
          >
            <a-button shape="circle" type="danger" icon="delete" />
          </a-popconfirm>
        </div>
      </a-list-item>
    </a-list>
  </a-spin>
</template>

<script>
import { api } from '@/api'

export default {
  name: 'DetailSettings',
  props: {
    resource: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      details: [],
      detailOptions: {},
      showAddDetail: false,
      disableSettings: false,
      newKey: '',
      newValue: '',
      loading: false,
      resourceType: 'UserVm',
      error: false
    }
  },
  watch: {
    resource: function (newItem, oldItem) {
      this.updateResource(newItem)
    }
  },
  mounted () {
    this.updateResource(this.resource)
  },
  methods: {
    filterOption (input, option) {
      return (
        option.componentOptions.children[0].text.toUpperCase().indexOf(input.toUpperCase()) >= 0
      )
    },
    updateResource (resource) {
      this.details = []
      if (!resource) {
        return
      }
      this.resource = resource
      this.resourceType = this.$route.meta.resourceType
      if (resource.details) {
        this.details = Object.keys(this.resource.details).map(k => {
          return { name: k, value: this.resource.details[k], edit: false }
        })
      }
      api('listDetailOptions', { resourcetype: this.resourceType, resourceid: this.resource.id }).then(json => {
        this.detailOptions = json.listdetailoptionsresponse.detailoptions.details
      })
      this.disableSettings = (this.$route.meta.name === 'vm' && this.resource.state !== 'Stopped')
    },
    showEditDetail (index) {
      this.details[index].edit = true
      this.details[index].originalValue = this.details[index].value
      this.$set(this.details, index, this.details[index])
    },
    hideEditDetail (index) {
      this.details[index].edit = false
      this.details[index].value = this.details[index].originalValue
      this.$set(this.details, index, this.details[index])
    },
    handleInputChange (val, index) {
      this.details[index].value = val
      this.$set(this.details, index, this.details[index])
    },
    onAddInputChange (val, obj) {
      this[obj] = val
    },
    isAdminOrOwner () {
      return ['Admin'].includes(this.$store.getters.userInfo.roletype) ||
        (this.resource.domainid === this.$store.getters.userInfo.domainid && this.resource.account === this.$store.getters.userInfo.account) ||
        this.resource.project && this.resource.projectid === this.$store.getters.project.id
    },
    runApi () {
      var apiName = ''
      if (this.resourceType === 'UserVm') {
        apiName = 'updateVirtualMachine'
      } else if (this.resourceType === 'Template') {
        apiName = 'updateTemplate'
      }
      if (!(apiName in this.$store.getters.apis)) {
        this.$notification.error({
          message: 'Failed to execute API: ' + apiName,
          description: 'User is not permitted to use the API'
        })
        return
      }

      const params = { id: this.resource.id }
      if (this.details.length === 0) {
        params.cleanupdetails = true
      } else {
        this.details.forEach(function (item, index) {
          params['details[0].' + item.name] = item.value
        })
      }
      this.loading = true
      api(apiName, params).then(json => {
        var details = {}
        if (this.resourceType === 'UserVm' && json.updatevirtualmachineresponse.virtualmachine.details) {
          details = json.updatevirtualmachineresponse.virtualmachine.details
        } else if (this.resourceType === 'Template' && json.updatetemplateresponse.template.details) {
          details = json.updatetemplateresponse.template.details
        }
        this.details = Object.keys(details).map(k => {
          return { name: k, value: details[k], edit: false }
        })
      }).catch(error => {
        this.$notifyError(error)
      }).finally(f => {
        this.loading = false
        this.showAddDetail = false
        this.newKey = ''
        this.newValue = ''
      })
    },
    addDetail () {
      if (this.newKey === '' || this.newValue === '') {
        this.error = 'Must provide a valid key and value for setting'
        return
      }
      this.error = false
      this.details.push({ name: this.newKey, value: this.newValue })
      this.runApi()
    },
    updateDetail (index) {
      this.runApi()
    },
    deleteDetail (index) {
      this.details.splice(index, 1)
      this.runApi()
    }
  }
}
</script>
