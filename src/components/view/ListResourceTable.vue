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
  <div>
    <a-input-search
      v-if="showSearch"
      style="width: 25vw;float: right;margin-bottom: 10px; z-index: 8"
      :placeholder="$t('label.search')"
      v-model="filter"
      @search="handleSearch" />

    <a-table
      size="small"
      :columns="fetchColumns()"
      :dataSource="items"
      :rowKey="item => item.id"
      :loading="loading"
      :pagination="false"
      @change="handleTableChange"
      @handle-search-filter="handleTableChange" >

      <template v-for="(column, index) in Object.keys(routerlinks({}))" :slot="column" slot-scope="text, item" >
        <span :key="index">
          <router-link :set="routerlink = routerlinks(item)" :to="{ path: routerlink[column] }" >{{ text }}</router-link>
        </span>
      </template>

      <template slot="state" slot-scope="text">
        <status :text="text ? text : ''" />{{ text }}
      </template>

      <template slot="status" slot-scope="text">
        <status :text="text ? text : ''" />{{ text }}
      </template>

    </a-table>

    <div style="display: block; text-align: right; margin-top: 10px;">
      <a-pagination
        size="small"
        :current="options.page"
        :pageSize="options.pageSize"
        :total="total"
        :showTotal="total => `Total ${total} ${$t('label.items')}`"
        :pageSizeOptions="['10', '20', '40']"
        @change="handleTableChange"
        @showSizeChange="handlePageSizeChange"
        showSizeChanger>
        <template slot="buildOptionText" slot-scope="props">
          <span>{{ props.value }} / {{ $t('label.page') }}</span>
        </template>
      </a-pagination>
    </div>
  </div>

</template>

<script>
import { api } from '@/api'
import Status from '@/components/widgets/Status'

export default {
  name: 'ListResourceTable',
  components: {
    Status
  },
  props: {
    resource: {
      type: Object,
      default: () => {}
    },
    apiName: {
      type: String,
      required: true
    },
    routerlinks: {
      type: Function,
      default: () => { return {} }
    },
    params: {
      type: Object,
      required: true
    },
    columns: {
      type: Array,
      required: true
    },
    showSearch: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      loading: false,
      items: [],
      total: 0,
      filter: '',
      options: {
        page: 1,
        pageSize: 10,
        keyword: null
      }
    }
  },
  watch: {
    resource (newItem, oldItem) {
      if (newItem !== oldItem) {
        this.fetchData()
      }
    },
    '$i18n.locale' (to, from) {
      if (to !== from) {
        this.fetchData()
      }
    }
  },
  mounted () {
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.loading = true
      var params = { ...this.params, ...this.options }
      params.listall = true
      params.response = 'json'
      params.details = 'min'
      api(this.apiName, params).then(json => {
        var responseName
        var objectName
        for (const key in json) {
          if (key.includes('response')) {
            responseName = key
            break
          }
        }
        for (const key in json[responseName]) {
          if (key === 'count') {
            this.total = json[responseName][key]
            continue
          }
          objectName = key
          break
        }
        this.items = json[responseName][objectName]
        if (!this.items || this.items.length === 0) {
          this.items = []
        }
      }).finally(() => {
        this.loading = false
      })
    },
    fetchColumns () {
      var columns = []
      for (const col of this.columns) {
        columns.push({
          dataIndex: col,
          title: this.$t('label.' + col),
          scopedSlots: { customRender: col }
        })
      }
      return columns
    },
    handleSearch (value) {
      this.filter = value
      this.options.page = 1
      this.options.pageSize = 10
      this.options.keyword = this.filter
      this.fetchData()
    },
    handleTableChange (page, pagesize) {
      this.options.page = page
      this.options.pageSize = pagesize
      this.fetchData()
    },
    handlePageSizeChange (page, pagesize) {
      this.options.page = 1
      this.options.pageSize = pagesize
      this.fetchData()
    }
  }
}
</script>
