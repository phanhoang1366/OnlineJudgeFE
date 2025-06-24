<template>
  <div class="view">
    <Panel :title="title">
      <el-form label-position="top">
        <el-row :gutter="20">
          <el-col :span="24">
            <el-form-item :label="$t('m.ContestTitle')" required>
              <el-input v-model="contest.title" :placeholder="$t('m.ContestTitle')"></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item :label="$t('m.ContestDescription')" required>
              <Simditor v-model="contest.description"></Simditor>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Contest_Start_Time')" required>
              <el-date-picker
                v-model="contest.start_time"
                type="datetime"
                :placeholder="$t('m.Contest_Start_Time')">
              </el-date-picker>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Contest_End_Time')" required>
              <el-date-picker
                v-model="contest.end_time"
                type="datetime"
                :placeholder="$t('m.Contest_End_Time')">
              </el-date-picker>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Contest_Password')">
              <el-input v-model="contest.password" :placeholder="$t('m.Contest_Password')"></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Contest_Rule_Type')">
              <el-radio class="radio" v-model="contest.rule_type" label="ACM" :disabled="disableRuleType">ACM</el-radio>
              <el-radio class="radio" v-model="contest.rule_type" label="OI" :disabled="disableRuleType">OI</el-radio>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Real_Time_Rank')">
              <el-switch
                v-model="contest.real_time_rank"
                active-color="#13ce66"
                inactive-color="#ff4949">
              </el-switch>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item :label="$t('m.Contest_Status')">
              <el-switch
                v-model="contest.visible"
                active-text=""
                inactive-text="">
              </el-switch>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <!-- Public ranking -->
            <el-form-item :label="$t('m.Public_Ranking')">
              <el-switch 
                v-model="contest.public_ranking"
                active-color="#13ce66"
                inactive-color="#ff4949">
              </el-switch>
            </el-form-item>
          </el-col>
          <el-col :span="16">
            <!-- Whitelist Usernames -->
            <el-form-item :label="$t('m.Whitelist_Usernames')">
              <el-switch 
                v-model="contest.whitelist_enabled">
              </el-switch>
              <el-button v-if="contest.whitelist_enabled" type="primary" @click="openDialogIfEnabled" style="margin-left: 10px;">
                {{ $t('m.Edit_Whitelist') }}
              </el-button>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item :label="$t('m.Allowed_IP_Ranges')">
              <div v-for="(range, index) in contest.allowed_ip_ranges" :key="index">
                <el-row :gutter="20" style="margin-bottom: 15px">
                  <el-col :span="8">
                    <el-input v-model="range.value" :placeholder="$t('m.CIDR_Network')"></el-input>
                  </el-col>
                  <el-col :span="10">
                    <el-button plain icon="el-icon-fa-plus" @click="addIPRange"></el-button>
                    <el-button plain icon="el-icon-fa-trash" @click="removeIPRange(range)"></el-button>
                  </el-col>
                </el-row>
              </div>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <save @click.native="saveContest"></save>
      <el-dialog :visible.sync="dialogVisible" :title="$t('m.Edit_Whitelist')" width="50%">
        <el-form>
          <el-form-item :label="$t('m.Whitelist_Upload_XLSX')">
            <el-alert
              title="Note: Uploading a whitelist will replace the existing one."
              type="warning"
              show-icon>
            </el-alert>
            <div style="margin: 10px 0; display: flex; align-items: center;">
              <el-upload
                ref="upload"
                action="/api/parse-whitelist"
                name="file"
                :on-success="uploadSucceeded"
                :on-error="uploadFailed"
                :accept="'.xlsx,.xls,.csv,.ods'"
                :with-credentials="true"
                :headers="uploadHeaders"
              >
                <el-button size="small" type="primary" icon="el-icon-fa-upload">Upload a spreadsheet...</el-button>
              </el-upload>
              <el-button type="primary" size="mini" icon="el-icon-download" @click="downloadSample" style="margin-left: 10px;">
                Sample
              </el-button>
            </div>
          </el-form-item>
          <el-table
            v-if="contest.whitelist_enabled"
            :data="contest.whitelistNames"
            :key="whitelistTableKey"
            style="width: 100%">
            <el-table-column prop="username" :label="$t('m.Username')"></el-table-column>
            <el-table-column prop="real_name" :label="$t('m.User_Real_Name')"></el-table-column>
          </el-table>
        </el-form>
      </el-dialog>
    </Panel>
  </div>
</template>

<script>
  import api from '../../api.js'
  import Simditor from '../../components/Simditor.vue'

  export default {
    name: 'CreateContest',
    components: {
      Simditor
    },
    data () {
      return {
        title: 'Create Contest',
        disableRuleType: false,
        dialogVisible: false,
        menuToggle: false,
        whitelistTableKey: 0,
        uploadHeaders: {
          'X-CSRFToken': this.getCSRFCookie()
        },
        contest: {
          title: '',
          description: '',
          start_time: '',
          end_time: '',
          rule_type: 'OI',
          password: '',
          real_time_rank: true,
          visible: true,
          public_ranking: false,
          whitelist_enabled: false,
          whitelist_users: [], // Used for storing user IDs
          whitelistNames: [], // Rendered in the dialog
          allowed_ip_ranges: [{
            value: ''
          }]
        }
      }
    },
    methods: {
      saveContest () {
        let funcName = this.$route.name === 'edit-contest' ? 'editContest' : 'createContest'
        let data = Object.assign({}, this.contest)
        let ranges = []
        for (let v of data.allowed_ip_ranges) {
          if (v.value !== '') {
            ranges.push(v.value)
          }
        }
        data.allowed_ip_ranges = ranges
        api[funcName](data).then(res => {
          this.$router.push({name: 'contest-list', query: {refresh: 'true'}})
        }).catch(() => {
        })
      },
      addIPRange () {
        this.contest.allowed_ip_ranges.push({value: ''})
      },
      removeIPRange (range) {
        let index = this.contest.allowed_ip_ranges.indexOf(range)
        if (index !== -1) {
          this.contest.allowed_ip_ranges.splice(index, 1)
        }
      },
      openDialogIfEnabled () {
        if (this.contest.whitelist_enabled) {
          this.dialogVisible = true
        }
      },
      uploadSucceeded (response) {
        if (response.error) {
          this.$error(response.data)
          return
        }
        if (!Array.isArray(response)) {
          this.$error(this.$t('m.Invalid_Whitelist_Format'))
          return
        }
        const data = response
        this.contest.whitelistNames = data.map(item => ({
          username: item.user && item.user.username ? item.user.username : '',
          real_name: item.real_name || (item.user && item.user.real_name) || ''
        }))
        this.contest.whitelist_users = data.map(item => item.user && item.user.id ? item.user.id : null).filter(id => id !== null)
        this.whitelistTableKey++
        this.$message.success(this.$t('m.Upload_Success'))
      },
      uploadFailed () {
        this.$error(this.$t('m.Upload_Failed'))
      },
      getCSRFCookie () {
        const match = document.cookie.match(/(?:^|; )csrftoken=([^;]*)/)
        return match ? decodeURIComponent(match[1]) : ''
      },
      downloadSample () {
        window.open('/public/xlsx/users.xlsx', '_blank')
      }
    },
    mounted () {
      if (this.$route.name === 'edit-contest') {
        this.title = 'Edit Contest'
        this.disableRuleType = true
        api.getContest(this.$route.params.contestId).then(res => {
          let data = res.data.data
          let ranges = []
          for (let v of data.allowed_ip_ranges) {
            ranges.push({value: v})
          }
          if (ranges.length === 0) {
            ranges.push({value: ''})
          }
          data.allowed_ip_ranges = ranges
          this.contest = data
          this.whitelistTableKey++
        }).catch(() => {
        })
      }
    },
    watch: {
      'contest.password' (newVal) {
        if (newVal && this.contest.whitelist_enabled) {
          this.contest.whitelist_enabled = false
        }
      },
      'contest.whitelist_enabled' (newVal) {
        if (newVal && this.contest.password) {
          this.contest.password = ''
        }
      }
    }
  }
</script>
