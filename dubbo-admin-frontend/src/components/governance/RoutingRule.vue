<!--
  - Licensed to the Apache Software Foundation (ASF) under one or more
  - contributor license agreements.  See the NOTICE file distributed with
  - this work for additional information regarding copyright ownership.
  - The ASF licenses this file to You under the Apache License, Version 2.0
  - (the "License"); you may not use this file except in compliance with
  - the License.  You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <v-container grid-list-xl fluid >
      <v-layout row wrap>
        <v-flex xs12 >
          <!--<v-card flat color="transparent">-->
            <!--<v-card-text>-->
              <!--<v-layout row wrap >-->
                <!--<v-text-field label="Search dubbo service"-->
                              <!--v-model="filter" clearable></v-text-field>-->
                <!--<v-btn @click="submit" color="primary" large>Search</v-btn>-->
              <!--</v-layout>-->
            <!--</v-card-text>-->
          <!--</v-card>-->
          <search v-model="filter" :submit="submit" label="Search dubbo service"></search>
        </v-flex>
      </v-layout>

    <v-flex lg12>
      <v-card>
        <v-toolbar flat color="transparent" class="elevation-0">
          <v-toolbar-title><span class="headline">Search Result</span></v-toolbar-title>
          <v-divider
            class="mx-2"
            inset
            vertical
          ></v-divider>
          <v-spacer></v-spacer>
          <v-btn outline color="primary" @click.stop="openDialog" class="mb-2">CREATE</v-btn>
        </v-toolbar>

        <v-card-text class="pa-0">
          <v-data-table
            :headers="headers"
            :items="routingRules"
            hide-actions
            class="elevation-0"
          >
            <template slot="items" slot-scope="props">
              <td class="text-xs-left">{{ props.item.service }}</td>
              <td class="text-xs-left">{{ props.item.group }}</td>
              <td class="text-xs-left">{{ props.item.priority }}</td>
              <td class="text-xs-left">{{ props.item.enabled }}</td>
              <td class="justify-center px-0">
                <v-tooltip bottom v-for="op in operations" :key="op.id">
                  <v-icon small class="mr-2" slot="activator" @click="itemOperation(op.icon(props.item), props.item)">
                    {{op.icon(props.item)}}
                  </v-icon>
                  <span>{{op.tooltip(props.item)}}</span>
                </v-tooltip>
              </td>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
    </v-flex>

    <v-dialog   v-model="dialog" width="800px" persistent >
      <v-card>
        <v-card-title class="justify-center">
          <span class="headline">Create New Routing Rule</span>
        </v-card-title>
        <v-card-text >
          <v-text-field
            label="Service Unique ID"
            hint="A service ID in form of service:version, version is optional"
            required
            v-model="service"
          ></v-text-field>
          <v-text-field
            label="Application Name"
            hint="Application name the service belongs to"
            required
            v-model="application"
          ></v-text-field>

          <v-subheader class="pa-0 mt-3">RULE CONTENT</v-subheader>
          <ace-editor v-model="ruleText" :readonly="readonly"></ace-editor>

        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" flat @click.native="closeDialog">Close</v-btn>
          <v-btn color="blue darken-1" flat @click.native="saveItem">Save</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="warn" persistent max-width="500px">
      <v-card>
        <v-card-title class="headline">{{this.warnTitle}}</v-card-title>
        <v-card-text >{{this.warnText}}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="green darken-1" flat @click.native="closeWarn">CANCLE</v-btn>
          <v-btn color="green darken-1" flat @click.native="deleteItem(warnStatus)">CONFIRM</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </v-container>

</template>
<script>
  import yaml from 'js-yaml'
  import AceEditor from '@/components/public/AceEditor'
  import Search from '@/components/public/Search'
  import {AXIOS} from '../http-common'
  export default {
    components: {
      AceEditor,
      Search
    },
    data: () => ({
      dropdown_font: [ 'Service', 'App', 'IP' ],
      ruleKeys: ['enabled', 'force', 'dynamic', 'runtime', 'group', 'version', 'rule', 'priority'],
      pattern: 'Service',
      filter: '',
      dialog: false,
      warn: false,
      updateId: -1,
      application: '',
      service: '',
      warnTitle: '',
      warnText: '',
      warnStatus: {},
      height: 0,
      operations: [
        {id: 0,
          icon: function (item) {
            return 'visibility'
          },
          tooltip: function (item) {
            return 'View'
          }},
        {id: 1,
          icon: function (item) {
            return 'edit'
          },
          tooltip: function (item) {
            return 'Edit'
          }},
        {id: 2,
          icon: function (item) {
            if (item.enabled) {
              return 'block'
            }
            return 'check_circle_outline'
          },
          tooltip: function (item) {
            if (item.enabled === true) {
              return 'Disable'
            }
            return 'Enable'
          }},
        {id: 3,
          icon: function (item) {
            return 'delete'
          },
          tooltip: function (item) {
            return 'Delete'
          }}
      ],
      routingRules: [
      ],
      template:
        'enabled: true/false\n' +
        'priority:\n' +
        'runtime: false/true\n' +
        'force: true/false\n' +
        'dynamic: true/false\n' +
        'conditions:\n' +
        ' - \'=> host != 172.22.3.91\'\n' +
        ' - \'host != 10.20.153.10,10.20.153.11 =>\'\n' +
        ' - \'host = 10.20.153.10,10.20.153.11 =>\'\n' +
        ' - \'application != kylin => host != 172.22.3.95,172.22.3.96\'\n' +
        ' - \'method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96\'',
      ruleText: '',
      readonly: false,
      headers: [
        {
          text: 'Service Name',
          value: 'service',
          align: 'left'
        },
        {
          text: 'Group',
          value: 'group',
          align: 'left'

        },
        {
          text: 'Priority',
          value: 'priority',
          sortable: false
        },
        {
          text: 'Enabled',
          value: 'enabled',
          sortable: false
        },
        {
          text: 'Operation',
          value: 'operation',
          sortable: false
        }
      ]
    }),
    methods: {
      submit: function () {
        this.search(this.filter, true)
      },
      search: function (filter, rewrite) {
        AXIOS.get('/routes/search?serviceName=' + filter)
          .then(response => {
            this.routingRules = response.data
            if (rewrite) {
              this.$router.push({path: 'routingRule', query: {service: filter}})
            }
          })
      },
      closeDialog: function () {
        this.ruleText = this.template
        this.updateId = -1
        this.service = ''
        this.dialog = false
        this.readonly = false
      },
      openDialog: function () {
        this.dialog = true
      },
      openWarn: function (title, text) {
        this.warnTitle = title
        this.warnText = text
        this.warn = true
      },
      closeWarn: function () {
        this.warnTitle = ''
        this.warnText = ''
        this.warn = false
      },
      saveItem: function () {
        let rule = yaml.safeLoad(this.ruleText)
        rule.service = this.service
        if (this.updateId !== -1) {
          rule.id = this.updateId
          AXIOS.post('/routes/update', rule)
            .then(response => {
              if (response.data) {
                this.search(this.service, true)
              }
              this.closeDialog()
            })
        } else {
          AXIOS.post('/routes/create', rule)
            .then(response => {
              if (response.data) {
                this.search(this.service, true)
                this.filter = this.service
              }
              this.closeDialog()
            })
        }
      },
      itemOperation: function (icon, item) {
        switch (icon) {
          case 'visibility':
            AXIOS.get('/routes/detail?id=' + item.id)
              .then(response => {
                let route = response.data
                this.service = route.service
                delete route.service
                this.ruleText = yaml.safeDump(route)
                this.readonly = true
                this.dialog = true
              })
            break
          case 'edit':
            let id = {}
            id.id = item.id
            AXIOS.get('/routes/detail?id=' + item.id)
              .then(response => {
                let route = response.data
                this.service = route.service
                delete route.service
                this.ruleText = yaml.safeDump(route)
                this.readonly = false
                this.dialog = true
                this.updateId = item.id
              })
            break
          case 'block':
            this.openWarn(' Are you sure to block Routing Rule', 'service: ' + item.service)
            this.warnStatus.operation = 'disable'
            this.warnStatus.id = item.id
            break
          case 'check_circle_outline':
            this.openWarn(' Are you sure to enable Routing Rule', 'service: ' + item.service)
            this.warnStatus.operation = 'enable'
            this.warnStatus.id = item.id
            break
          case 'delete':
            this.openWarn(' Are you sure to Delete Routing Rule', 'service: ' + item.service)
            this.warnStatus.operation = 'delete'
            this.warnStatus.id = item.id
        }
      },
      setHeight: function () {
        this.height = window.innerHeight * 0.5
      },
      deleteItem: function (warnStatus) {
        if (warnStatus.operation === 'delete') {
          let id = {}
          id.id = warnStatus.id
          AXIOS.post('/routes/delete', id)
            .then(response => {
              this.warn = false
              this.search(this.filter, false)
            })
        } else if (warnStatus.operation === 'disable') {
          let id = {}
          id.id = warnStatus.id
          AXIOS.post('/routes/disable', id)
            .then(response => {
              this.warn = false
              this.search(this.filter, false)
            })
        } else if (warnStatus.operation === 'enable') {
          let id = {}
          id.id = warnStatus.id
          AXIOS.post('/routes/enable', id)
            .then(response => {
              this.warn = false
              this.search(this.filter, false)
            })
        }
      }
    },
    created () {
      this.setHeight()
    },
    mounted: function () {
      this.ruleText = this.template
      let query = this.$route.query
      let service = ''
      Object.keys(query).forEach(function (key) {
        if (key === 'service') {
          service = query[key]
        }
      })
      if (service !== '') {
        this.filter = service
        this.search(service, false)
      }
    }

  }
</script>