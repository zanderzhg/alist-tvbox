<template>
  <div class="list">
    <h1>阿里账号列表</h1>
    <el-row justify="end">
      <el-button @click="load">刷新</el-button>
      <el-button type="primary" @click="handleAdd">添加</el-button>
    </el-row>
    <div class="space"></div>

    <el-table :data="accounts" border style="width: 100%">
<!--      <el-table-column prop="id" label="ID" sortable width="70"/>-->
      <el-table-column prop="nickname" label="昵称" sortable width="180"/>
      <el-table-column prop="autoCheckin" label="自动签到" width="90">
        <template #default="scope">
          <el-icon v-if="scope.row.autoCheckin">
            <Check/>
          </el-icon>
          <el-icon v-else>
            <Close/>
          </el-icon>
        </template>
      </el-table-column>
      <el-table-column prop="checkinDays" label="签到次数" width="90">
        <template #default="scope">
          {{ scope.row.checkinDays }}
          <span class="divider"></span>
          <el-button link @click="loadTimeline(scope.row.id)">
            <el-icon><Calendar /></el-icon>
          </el-button>
        </template>
      </el-table-column>
      <el-table-column prop="checkinTime" label="上次签到时间">
        <template #default="scope">
          {{ formatTime(scope.row.checkinTime) }}
        </template>
      </el-table-column>
      <el-table-column prop="showMyAli" label="加载我的云盘？" width="150">
        <template #default="scope">
          <el-icon v-if="scope.row.showMyAli">
            <Check/>
          </el-icon>
          <el-icon v-else>
            <Close/>
          </el-icon>
        </template>
      </el-table-column>
      <el-table-column prop="master" label="主账号？" width="120">
        <template #default="scope">
          <el-icon v-if="scope.row.master">
            <Check/>
          </el-icon>
          <el-icon v-else>
            <Close/>
          </el-icon>
        </template>
      </el-table-column>
      <el-table-column prop="master" label="开启代理？" width="120">
        <template #default="scope">
          <el-icon v-if="scope.row.useProxy">
            <Check/>
          </el-icon>
          <el-icon v-else>
            <Close/>
          </el-icon>
        </template>
      </el-table-column>
      <el-table-column prop="concurrency" label="线程数" width="110"/>
      <el-table-column fixed="right" label="操作" width="200">
        <template #default="scope">
          <el-button link type="primary" size="small" @click="showDetails(scope.row)">详情</el-button>
          <el-button link type="danger" size="small" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-dialog v-model="formVisible" :title="dialogTitle" width="60%">
      <el-form :model="form" label-width="auto">
        <el-form-item label="阿里refresh token" required>
          <el-input v-model="form.refreshToken" maxlength="128" placeholder="长度32位" autocomplete="off"/>
          <a href="https://ali.har01d.org/" target="_blank">获取阿里token</a>
          <a href="https://api.oplist.org/" target="_blank" class="hint">获取阿里token</a>
        </el-form-item>
        <el-form-item label="开放refresh token" required>
          <el-input v-model="form.openToken" type="textarea" rows="3" minlength="256" placeholder="长度280位"
                    autocomplete="off"/>
          <div v-if="tokenUrl.includes('har01d.org')">
            har01d:<a href="https://ali.har01d.org/authorize" title="需要选择har01d的认证URL" target="_blank">获取开放token</a>
          </div>
          <div class="hint" v-if="tokenUrl.includes('ycyup.cn')">
            ycyup:<a href="https://ycyup.cn/alipan/authorize" title="需要选择ycyup的认证URL" target="_blank">获取开放token</a>
          </div>
          <div class="hint">
            TV Token:<a href="javascript:void(0)" @click.stop="getQr">扫码获取</a>
          </div>
        </el-form-item>
        <el-form-item label="加载我的云盘" v-if="form.openToken">
          <el-switch
            v-model="form.showMyAli"
            inline-prompt
            active-text="开启"
            inactive-text="关闭"
          />
        </el-form-item>
        <el-form-item label="主账号">
          <el-switch
            v-model="form.master"
            inline-prompt
            active-text="是"
            inactive-text="否"
          />
          <span class="hint">主账号用来观看分享。</span>
        </el-form-item>
        <el-form-item label="加速代理">
          <el-switch
            v-model="form.useProxy"
            inline-prompt
            active-text="开启"
            inactive-text="关闭"
          />
          <span class="hint">服务端多线程加速，网页播放强制开启</span>
        </el-form-item>
        <el-form-item label="代理线程数">
          <el-input-number :min="1" :max="16" v-model="form.concurrency"/>
        </el-form-item>
        <el-form-item label="分片大小">
          <el-input-number :min="64" :max="4096" v-model="form.chunkSize"/>
        </el-form-item>
        <el-form-item label="自动签到">
          <el-switch
            v-model="form.autoCheckin"
            inline-prompt
            active-text="开启"
            inactive-text="关闭"
          />
        </el-form-item>
      </el-form>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="handleCancel">取消</el-button>
        <el-button type="primary" @click="handleConfirm">{{ updateAction ? '更新' : '添加' }}</el-button>
      </span>
      </template>
    </el-dialog>

    <el-dialog v-model="dialogVisible" title="删除阿里账号" width="30%">
      <p>是否删除阿里账号 - {{ form.id }}</p>
      <p>{{ form.nickname }}</p>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="danger" @click="deleteAccount">删除</el-button>
      </span>
      </template>
    </el-dialog>

    <el-dialog v-model="detailVisible" title="阿里账号详情" width="60%">
      <el-form :model="form" label-width="auto">
        <el-form-item v-if="form.accessTokenOpen" prop="accessTokenOpen" label="开放access token">
          <el-input v-model="form.accessTokenOpen" maxlength="128" readonly/>
          <span class="hint">创建时间： {{ formatTime(iat[1]) }}</span>
          <span class="hint">更新时间： {{ formatTime(form.accessTokenOpenTime) }}</span>
          <span class="hint">过期时间： {{ formatTime(exp[1]) }}</span>
        </el-form-item>
        <el-form-item prop="refreshToken" label="阿里refresh token" required>
          <el-input v-model="form.refreshToken" maxlength="128" placeholder="长度32位"/>
          <a href="https://ali.har01d.org/" target="_blank">获取阿里token</a>
          <a href="https://api.oplist.org/" target="_blank" class="hint">获取阿里token</a>
          <span class="hint">更新时间： {{ formatTime(form.refreshTokenTime) }}</span>
        </el-form-item>
        <el-form-item prop="openToken" label="开放refresh token" required>
          <el-input v-model="form.openToken" type="textarea" rows="4" minlength="256" placeholder="长度280位"/>
          <div v-if="tokenUrl.includes('har01d.org')">
            har01d:<a href="https://ali.har01d.org/authorize" title="需要选择har01d的认证URL" target="_blank">获取开放token</a>
          </div>
          <div class="hint" v-if="tokenUrl.includes('ycyup.cn')">
            ycyup:<a href="https://ycyup.cn/alipan/authorize" title="需要选择ycyup的认证URL" target="_blank">获取开放token</a>
          </div>
          <div class="hint">
            TV Token:<a href="javascript:void(0)" @click.stop="getQr">扫码获取</a>
          </div>
          <span class="hint">创建时间： {{ formatTime(iat[2]) }}</span>
          <span class="hint">更新时间： {{ formatTime(form.openTokenTime) }}</span>
          <span class="hint">过期时间： {{ formatTime(exp[2]) }}</span>
        </el-form-item>
        <el-form-item label="加载我的云盘" v-if="form.openToken">
          <el-switch
            v-model="form.showMyAli"
            inline-prompt
            active-text="加载"
            inactive-text="关闭"
          />
        </el-form-item>
        <el-form-item label="主账号">
          <el-switch
            v-model="form.master"
            inline-prompt
            active-text="是"
            inactive-text="否"
          />
          <span class="hint">主账号用来观看分享。</span>
        </el-form-item>
        <el-form-item label="自动签到">
          <el-switch
            v-model="form.autoCheckin"
            inline-prompt
            active-text="开启"
            inactive-text="关闭"
          />
        </el-form-item>
        <el-form-item label="加速代理">
          <el-switch
            v-model="form.useProxy"
            inline-prompt
            active-text="开启"
            inactive-text="关闭"
          />
          <span class="hint">服务端多线程加速，网页播放强制开启</span>
        </el-form-item>
        <el-form-item label="代理线程数">
          <el-input-number :min="1" :max="16" v-model="form.concurrency"/>
        </el-form-item>
        <el-form-item label="分片大小">
          <el-input-number :min="64" :max="4096" v-model="form.chunkSize"/>
        </el-form-item>
        <el-form-item label="上次签到时间" v-if="form.checkinTime">
          <el-input :model-value="formatTime(form.checkinTime)" readonly/>
          <span class="hint" v-if="form.checkinDays">{{ form.nickname }} 本月签到{{ form.checkinDays }}次</span>
        </el-form-item>
        <el-checkbox v-model="forceCheckin" label="强制签到"/>
      </el-form>
      <template #footer>
          <span class="dialog-footer">
            <el-button @click="detailVisible = false">取消</el-button>
            <el-button type="success" @click="checkin">签到</el-button>
            <el-button type="primary" @click="handleConfirm">更新</el-button>
          </span>
      </template>
    </el-dialog>

    <el-dialog v-model="alistVisible" title="更新成功" width="40%">
      <p>需要重启AList服务后才能生效</p>
      <p>是否重启AList服务？</p>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="alistVisible = false">取消</el-button>
        <el-button type="danger" @click="restartAList">重启</el-button>
      </span>
      </template>
    </el-dialog>

    <el-dialog v-model="timelineVisible" title="签到日志" width="60%">
      <el-timeline>
        <el-timeline-item
          v-for="(activity, index) in activities"
          :key="index"
          :type="activity.status!='end'?'primary':''"
          :hollow="activity.status!='verification'"
          :timestamp="activity.date"
        >
          {{ activity.name }}
        </el-timeline-item>
      </el-timeline>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="timelineVisible = false">关闭</el-button>
      </span>
      </template>
    </el-dialog>

    <el-dialog v-model="qrVisible" title="会员TV Token扫码" width="50%">
      <img alt="qr" :src="'data:image/png;base64,'+ base64QrCode" style="width: 500px;">
      <el-form-item>
        <el-button type="primary" @click="checkQr">我已经授权</el-button>
      </el-form-item>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="qrVisible = false">取消</el-button>
      </span>
      </template>
    </el-dialog>

    <div class="divider"></div>

    <PikPakView></PikPakView>

    <div class="divider"></div>

    <pan-account-view></pan-account-view>
  </div>
</template>

<script setup lang="ts">
import {onMounted, ref} from 'vue'
import {Calendar, Check, Close} from '@element-plus/icons-vue'
import axios from "axios"
import {ElMessage} from "element-plus";
import {store} from "@/services/store";
import router from "@/router";
import PikPakView from '@/views/PikPakView.vue'
import PanAccountView from "@/views/PanAccountView.vue";

const iat = ref([0])
const exp = ref([0])
const activities = ref<any[]>([])
const forceCheckin = ref(false)
const updateAction = ref(false)
const dialogTitle = ref('')
const sid = ref('')
const code = ref('')
const tokenUrl = ref('')
const base64QrCode = ref('')
const accounts = ref([])
const formVisible = ref(false)
const dialogVisible = ref(false)
const qrVisible = ref(false)
const detailVisible = ref(false)
const alistVisible = ref(false)
const timelineVisible = ref(false)
const form = ref({
  id: 0,
  nickname: '',
  refreshToken: '',
  openToken: '',
  accessToken: '',
  accessTokenOpen: '',
  autoCheckin: true,
  showMyAli: false,
  useProxy: false,
  master: false,
  refreshTokenTime: '',
  openTokenTime: '',
  accessTokenTime: '',
  accessTokenOpenTime: '',
  checkinTime: '',
  checkinDays: 1,
  concurrency: 2,
  chunkSize: 256,
})

const formatTime = (value: string | number) => {
  return new Date(value).toLocaleString('zh-cn')
}

const showDetails = (data: any) => {
  form.value = Object.assign({}, data)
  updateAction.value = true
  if (form.value.accessToken) {
    let details = JSON.parse(atob(form.value.accessToken.split('.')[1]))
    iat.value[0] = details.iat * 1000
    exp.value[0] = details.exp * 1000
  }
  if (form.value.accessTokenOpen) {
    let details = JSON.parse(atob(form.value.accessTokenOpen.split('.')[1]))
    iat.value[1] = details.iat * 1000
    exp.value[1] = details.exp * 1000
  }
  if (form.value.openToken) {
    let details = JSON.parse(atob(form.value.openToken.split('.')[1]))
    iat.value[2] = details.iat * 1000
    exp.value[2] = details.exp * 1000
  }
  detailVisible.value = true
}

const checkin = () => {
  axios.post('/api/ali/accounts/' + form.value.id + '/checkin?force=' + forceCheckin.value).then(({data}) => {
    form.value.checkinTime = data.checkinTime
    form.value.checkinDays = data.signInCount
    form.value.nickname = data.nickname
    forceCheckin.value = false
    ElMessage.success('签到成功, 本月累计' + data.signInCount + '天')
    load()
  })
}

const getQr = () => {
  axios.post('/ali/auth/qr').then(({data}) => {
    base64QrCode.value = data.img
    sid.value = data.sid
    qrVisible.value = true
  })
}

const checkQr = () => {
  axios.get('/ali/auth/qr?sid=' + sid.value).then(({data}) => {
    code.value = data
    form.value.accessTokenOpen = data.access_token
    form.value.openToken = data.refresh_token
    qrVisible.value = false
  })
}

const handleAdd = () => {
  dialogTitle.value = '添加阿里账号'
  updateAction.value = false
  form.value = {
    id: 0,
    nickname: '',
    refreshToken: '',
    openToken: '',
    accessToken: '',
    accessTokenOpen: '',
    autoCheckin: true,
    showMyAli: false,
    useProxy: false,
    master: false,
    refreshTokenTime: '',
    openTokenTime: '',
    accessTokenTime: '',
    accessTokenOpenTime: '',
    checkinTime: '',
    checkinDays: 1,
    concurrency: 2,
    chunkSize: 256,
  }
  formVisible.value = true
}

const handleDelete = (data: any) => {
  form.value = data
  dialogVisible.value = true
}

const deleteAccount = () => {
  dialogVisible.value = false
  axios.delete('/api/ali/accounts/' + form.value.id).then(() => {
    load()
  })
}

const handleCancel = () => {
  formVisible.value = false
}

const handleConfirm = () => {
  const url = updateAction.value ? '/api/ali/accounts/' + form.value.id : '/api/ali/accounts'
  axios.post(url, form.value).then((response) => {
    detailVisible.value = false
    if (accounts.value.length === 0) {
      if (store.aListStatus) {
        ElMessage.success('添加成功')
      } else {
        ElMessage.success('添加成功，AList服务重启中。')
        setTimeout(() => router.push('/wait'), 3000)
      }
    } else {
      if (response.headers['alist_restart_required']) {
        ElMessage.success('更新成功，需要重启AList生效')
        alistVisible.value = true
      } else {
        ElMessage.success('更新成功')
      }
    }
    formVisible.value = false
    load()
  })
}

const restartAList = () => {
  axios.post('/api/alist/restart').then(() => {
    alistVisible.value = false
    ElMessage.success('AList重启中')
    setTimeout(() => router.push('/wait'), 1000)
  })
}

const loadTimeline = (id: number) => {
  axios.get('/api/ali/accounts/' + id + '/checkin').then(({data}) => {
    activities.value = data
    timelineVisible.value = true
  })
}

const load = () => {
  axios.get('/api/ali/accounts').then(({data}) => {
    accounts.value = data
  })
}

onMounted(() => {
  load()
  axios.get('/api/settings/open_token_url').then(({data}) => {
    tokenUrl.value = data.value
  })
})
</script>

<style scoped>
.divider {
  margin: 30px 0;
}

.space {
  margin-bottom: 6px;
}

.json pre {
  height: 600px;
  overflow: scroll;
}
</style>
