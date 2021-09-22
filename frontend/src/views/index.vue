<template>
  <div class="content">
    <div class="card">
      <div class="card-header">
        <p class="card-title">个人中心</p>
      </div>
      <div class="card-body">
        <p>昵称：{{ nickName }}</p>
        <p>更新时间：{{ timestamp }}</p>
      </div>
      <div class="card-footer">
        <el-button size="small" auto @click="logout">退出登录</el-button>
        <el-button type="danger" size="small" auto @click="delAccount">删除CK</el-button>
      </div>
    </div>
    
    <div class="card">
      <div class="card-header">
          <p class="card-title">WSCK 录入</p>
            <div class="card-body text-base leading-6">
              <b>wskey有效期长达一年，请联系管理员确认使用，慎重！</b>
              <p>删WSCK在下方。</p>
              <b>也可以保持pin不变，随意更改wskey，等同于删除WSCK。改密码解决一切CK泄露问题。</b>
              <p>用户须手动提取pin和wskey，格式如："pin=xxxxxx;wskey=xxxxxxxxxx;"。</p>
              <p class="card-subtitle">——IOS用户手机抓包APP&emsp;<a style="" href="https://apps.apple.com/cn/app/stream/id1312141691" target="_blank" id="downiOSApp">点击跳转安装</a> </p>
              <p class="card-subtitle">——在api.m.jd.com域名下找POST请求大概率能找到wskey。</p>
              <p class="card-subtitle">wskey在录入后立马上线，系统会在指定时间检查wskey，有效则自动转换出cookie登录</p>
              <p class="card-subtitle">cookie失效后，也会在系统设定的指定时间内自动转换出新的cookie，实现一次录入长期有效</p>
              <b>wskey会随着京东app的退出登录和更改密码而失效，清楚app数据或者卸载软件不会影响。</b>
            </div>
      </div>
      <div class="card-body text-center">
        <el-input v-model="jdwsck" placeholder="pin=xxxxxx;wskey=xxxxxxxxxx;" size="small" clearable class="my-4 w-full" />
      </div>
      <div class="card-footer">
        <el-button type="success" size="small" auto @click="WSCKLogin">重新录入</el-button>
        <el-button type="danger" size="small" auto @click="delWSCKAccount">删除WSCK</el-button>
      </div>
    </div>
    
    <div class="card">
      <div class="card-header">
        <p class="card-title">修改备注（CK和WSCK同步）</p>
      </div>
      <div class="card-body text-center">
        <el-input v-model="remark" size="small" clearable class="my-4 w-full" />
      </div>
      <div class="card-footer">
        <el-button type="success" size="small" auto @click="changeremark">修改</el-button>
      </div>
    </div>
    
    <div class="card">
      <div class="card-header">
        <p class="card-title">常见活动位置</p>
        <span class="card-subtitle">下面是一些常见活动的位置</span>
      </div>
      <div class="card-body">
        <ul>
          <li
            v-for="(item, index) in activity"
            :key="index"
            class="leading-normal"
          >
            <span>{{ item.name }}：</span>
            <span class="pr-2">{{ item.address }}</span>
            <a
              v-if="item.href"
              class="text-blue-400"
              href="#"
              @click="openUrlWithJD(item.href)"
              >直达链接</a
            >
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import { getUserInfoAPI, delAccountAPI, remarkupdateAPI, WSCKLoginAPI, WSCKDelaccountAPI, remarkupdateWSCKAPI } from '@/api/index'
import { onMounted, reactive, toRefs } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { ElMessage } from 'element-plus'
export default {
  setup() {
    const router = useRouter()
    const route = useRoute()

    let data = reactive({
      remark: '',
      jdwsck: undefined,
      nickName: undefined,
      timestamp: undefined,
    })

    const getInfo = async () => {
      const eid = localStorage.getItem('eid')
      const wseid = localStorage.getItem('wseid')
      if (!eid && !wseid) {
        logout()
        return
      }
      if (eid) {
        const userInfo = await getUserInfoAPI(eid)
        if (!userInfo) {
          ElMessage.error('获取用户CK信息失败，请重重新登录')
          logout()
          return
        }
        data.nickName = userInfo.data.nickName
        data.timestamp = new Date(userInfo.data.timestamp).toLocaleString()
      }
      
      if (wseid) {
        const userInfo = await getWSCKUserinfoAPI(wseid)
        if (!userInfo) {
          ElMessage.error('获取用户WSCK信息失败，请重重新登录')
          logout()
          return
        }
        data.nickName = userInfo.data.nickName
        data.timestamp = new Date(userInfo.data.timestamp).toLocaleString()
      }
    }

    onMounted(getInfo)

    const logout = () => {
      localStorage.removeItem('eid')
      localStorage.removeItem('wseid')
      router.push('/login')
    }

    const delAccount = async () => {
      const eid = localStorage.getItem('eid')
      const body = await delAccountAPI({ eid })
      if (body.code !== 200) {
        ElMessage.error(body.message)
      } else {
        ElMessage.success(body.message)
        setTimeout(() => {
          logout()
        }, 1000)
      }
    }
    
    const changeremark = async () => {
      const eid = localStorage.getItem('eid')
      const wseid = localStorage.getItem('wseid')
      const remark = data.remark
      if (eid) {
        const body = await remarkupdateAPI({ eid, remark })
        if (body.code !== 200) {
          ElMessage.success(body.message)
        } else {
          ElMessage.error(body.message)
        }
      }
      if (wseid) {
        const wsbody = await remarkupdateWSCKAPI({ wseid, remark })
        if (wsbody.code !== 200) {
          ElMessage.success(wsbody.message)
        } else {
          ElMessage.error(wsbody.message)
        }
      }
    }
    
    const WSCKLogin = async () => {
      const wskey =
        data.jdwsck.match(/wskey=(.*?);/) &&
        data.jdwsck.match(/wskey=(.*?);/)[1]
      const pin =
        data.jdwsck.match(/pin=(.*?);/) &&
        data.jdwsck.match(/pin=(.*?);/)[1]
      if (wskey && pin) {
        const body = await WSCKLoginAPI({ wskey: wskey, pin: pin })
        if (body.data.wseid) {
          localStorage.setItem('wseid', body.data.wseid)
          ElMessage.success(body.message)
        } else {
          ElMessage.error(body.message || 'wskey 解析失败，请检查后重试！')
        }
      } else {
        ElMessage.error('wskey 解析失败，请检查后重试！')
      }
    }
    
    const delWSCKAccount = async () => {
      const wseid = localStorage.getItem('wseid')
      const body = await WSCKDelaccountAPI({ wseid })
      if (body.code !== 200) {
        ElMessage.error(body.message)
      } else {
        ElMessage.success(body.message)
        setTimeout(() => {
          logout()
        }, 1000)
      }
    }
    
    const openUrlWithJD = (url) => {
      const params = encodeURIComponent(
        `{"category":"jump","des":"m","action":"to","url":"${url}"}`
      )
      window.location.href = `openapp.jdmobile://virtual?params=${params}`
      console.log(window.location.href)
    }

    const activity = [
      {
        name: '玩一玩（可找到大多数活动）',
        address: '京东 APP 首页-频道-边玩边赚',
        href: 'https://funearth.m.jd.com/babelDiy/Zeus/3BB1rymVZUo4XmicATEUSDUgHZND/index.html',
      },
      {
        name: '宠汪汪',
        address: '京东APP-首页/玩一玩/我的-宠汪汪',
      },
      {
        name: '东东萌宠',
        address: '京东APP-首页/玩一玩/我的-东东萌宠',
      },
      {
        name: '东东农场',
        address: '京东APP-首页/玩一玩/我的-东东农场',
      },
      {
        name: '东东工厂',
        address: '京东APP-首页/玩一玩/我的-东东工厂',
      },
      {
        name: '东东超市',
        address: '京东APP-首页/玩一玩/我的-东东超市',
      },
      {
        name: '领现金',
        address: '京东APP-首页/玩一玩/我的-领现金',
      },
      {
        name: '东东健康社区',
        address: '京东APP-首页/玩一玩/我的-东东健康社区',
      },
      {
        name: '京喜农场',
        address: '京喜APP-我的-京喜农场',
      },
      {
        name: '京喜牧场',
        address: '京喜APP-我的-京喜牧场',
      },
      {
        name: '京喜工厂',
        address: '京喜APP-我的-京喜工厂',
      },
      {
        name: '京喜财富岛',
        address: '京喜APP-我的-京喜财富岛',
      },
      {
        name: '京东极速版红包',
        address: '京东极速版APP-我的-红包',
      },
    ]

    return {
      ...toRefs(data),
      activity,
      getInfo,
      logout,
      delAccount,
      changeremark,
      WSCKLogin,
      delWSCKAccount,
      openUrlWithJD,
    }
  },
}
</script>

<style scoped>
/*没被访问过之前*/
 a:link{
            color: #B321FF;
        }
        /*默认*/
 a{
            color: #EECDFF;
        }
        /*鼠标掠过*/
 a:hover{
            color: red;
        }
</style>
