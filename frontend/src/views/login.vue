<template>
  <div class="content">
    <div class="card">
      <div class="card-header">
        <div class="flex items-center justify-between">
          <p class="card-title">Ninja提醒您</p>
        </div>
      </div>
      <div class="card-body text-base leading-6">
        <p>为了您的财产安全请关闭免密支付以及打开支付验密（京东-设置-支付设置-支付验密设置）。</p>
        <p>建议京东账户绑定微信以保证提现能到账。</p>
        <p>由于京东异地登录限制，扫码获取cookie只有2小时有效期，因此暂时关闭扫码功能，现需手动抓取Cookie。</p>
        <p>且有效期不长，平均3-5天，因此需要及时更新。</p>
        <b>安全起见，WSCK可以在CK登录后录入，期限半永久。</b>
      </div>
      <div class="card-footet"></div>
    </div>

    <div v-if="showQR" class="card">
      <div class="card-header">
        <div class="flex items-center justify-between">
          <p class="card-title">扫码登录</p>
          <span class="ml-2 px-2 py-1 bg-gray-200 rounded-full font-normal text-xs">余量：{{ marginCount }}</span>
        </div>
        <span class="card-subtitle">
          请点击下方按钮登录，点击按钮后回到本网站查看是否登录成功，京东的升级提示不用管。
        </span>
      </div>
      <div class="card-body text-center">
        <div v-if="!qrCodeVisibility" class="flex flex-col w-48 m-auto mt-4">
          <el-button type="primary" round @click="showQrcode"
            >扫描二维码登录</el-button
          >
          <el-button class="mt-4 ml-0" type="primary" round @click="jumpLogin"
            >跳转到京东 App 登录</el-button
          >
        </div>
        <img v-else :src="QRCode" :width="256" class="m-auto" />
      </div>
      <div class="card-footer"></div>
    </div>

    <div v-if="showWSCK" class="card">
      <div class="card-header">
        <div class="flex items-center justify-between">
          <p class="card-title">WSCK 录入</p>
          <span class="ml-2 px-2 py-1 bg-gray-200 rounded-full font-normal text-xs">余量：{{ marginWSCKCount }}</span>
        </div>
        <div class="card-body text-base leading-6">
          <b>wskey有效期长达一年，请联系管理员确认使用（删不掉，慎用）</b>
          <p>用户须手动提取pin和wskey，格式如："pt_pin=xxxxxx;wskey=xxxxxxxxxx;"。</p>
          <p class="card-subtitle">——IOS用户手机抓包APP&emsp;<a style="" href="https://apps.apple.com/cn/app/stream/id1312141691" target="_blank" id="downiOSApp">点击跳转安装</a> </p>
          <p class="card-subtitle">——在api.m.jd.com域名下找POST请求大概率能找到wskey。</p>
          <p class="card-subtitle">wskey在录入后立马上线，系统会在指定时间检查wskey，有效则自动转换出cookie登录</p>
          <p class="card-subtitle">cookie失效后，也会在系统设定的指定时间内自动转换出新的cookie，实现一次录入长期有效</p>
          <b>wskey会随着京东app的退出登录和更改密码而失效，清楚app数据或者卸载软件不会影响。</b>
        </div>
        <span class="card-subtitle"> 请在下方输入您的 WSCK  </span>
      </div>
      <div class="card-body text-center">
        <el-input v-model="jdwsck" placeholder="pin=xxxxxx;wskey=xxxxxxxxxx;" size="small" clearable class="my-4 w-full" />
        <el-button type="primary" size="small" round @click="WSCKLogin">录入</el-button>
      </div>
      <div class="card-footet"></div>
    </div>

    <div v-if="showCK" class="card">
      <div class="card-header">
        <div class="flex items-center justify-between">
          <p class="card-title">CK 登录</p>
          <span class="ml-2 px-2 py-1 bg-gray-200 rounded-full font-normal text-xs">余量：{{ marginCount }}</span>
        </div>
        <div class="card-body text-base leading-6">
          <p>PC用户建议使用开源工具<a style="" href="https://github.com/Waikkii/JD_Get_Cookie" target="_blank" id="waikiki">JD_Get_Cookie</a>获取cookie并在下方填写。</p>
          <p>手机用户可以使用Alook浏览器登录<a style="" href="https://m.jd.com/" target="_blank" id="jd">JD官网</a>，并在菜单-工具箱-开发者工具-Cookies中获取（Android和iPhone通用）。</p>
          <p>另外也可以使用抓包工具（iPhone：Stream，Android：HttpCanary）抓取京东app的ck，要注意pt_key和pt_pin字段是以app_open开头的。</p>
          <p>cookie直接填入输入框即可，Ninja会自动正则提取pt_key和pt_pin。</p>
        </div>
        <span class="card-subtitle"> 请在下方输入您的 cookie 登录。 </span>
      </div>
      <div class="card-body text-center">
        <el-input v-model="cookie" size="small" clearable class="my-4 w-full" />
        <el-button type="primary" size="small" round @click="CKLogin">登录</el-button>
      </div>
      <div class="card-footet"></div>
    </div>
  </div>
</template>

<script>
import { onMounted, reactive, toRefs } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { ElMessage } from 'element-plus'
import {
  getInfoAPI,
  getQrcodeAPI,
  CKLoginAPI,
  checkLoginAPI,
  WSCKLoginAPI,
} from '@/api/index'

export default {
  setup() {
    const router = useRouter()
    const route = useRoute()

    let data = reactive({
      marginCount: 0,
      allowAdd: true,
      cookie: '',
      QRCode: undefined,
      qrCodeVisibility: false,
      token: undefined,
      okl_token: undefined,
      cookies: undefined,
      timer: undefined,
      waitLogin: false,
      //
      marginWSCKCount: 0,
      allowWSCKAdd: true,
      jdwsck: undefined,
      showQR:false,
      showWSCK:false,
      showCK:true,

    })

    const getInfo = async () => {
      const info = (await getInfoAPI()).data
      data.marginCount = info.marginCount
      data.allowAdd = info.allowAdd
      data.marginWSCKCount = info.marginWSCKCount
      data.allowWSCKAdd = info.allowWSCKAdd
      data.showQR = info.showQR
      data.showWSCK = info.showWSCK
      data.showCK = info.showCK

    }

    const getQrcode = async () => {
      // 增加扫码是否禁用判断
      if (this.showQR) {
        try {
          const body = await getQrcodeAPI()
          data.token = body.data.token
          data.okl_token = body.data.okl_token
          data.cookies = body.data.cookies
          data.QRCode = body.data.QRCode
          if (data.QRCode) {
            // data.qrCodeVisibility = true
            data.waitLogin = true
            clearInterval(data.timer) // 清除定时器
            data.timer = setInterval(ckeckLogin, 3000) // 设置定时器
          }
        } catch (e) {
          console.error(e)
          ElMessage.error('生成二维码失败！请重试或放弃')
        }
      } else {
        ElMessage.warning('扫码已禁用请手动抓包')
      }

    }

    const showQrcode = async () => {
      data.qrCodeVisibility = true
    }

    const jumpLogin = async () => {
      const href = `openapp.jdmobile://virtual/ad?params={"category":"jump","des":"ThirdPartyLogin","action":"to","onekeylogin":"return","url":"https://plogin.m.jd.com/cgi-bin/m/tmauth?appid=300&client_type=m&token=${data.token}","authlogin_returnurl":"weixin://","browserlogin_fromurl":"${window.location.host}"}`
      window.location.href = href
    }

    const ckeckLogin = async () => {
      try {
        const body = await checkLoginAPI({
          token: data.token,
          okl_token: data.okl_token,
          cookies: data.cookies,
        })

        switch (body?.data.errcode) {
          case 0:
            localStorage.setItem('eid', body.data.eid)
            ElMessage.success(body.message)
            clearInterval(data.timer)
            router.push('/')
            break
          case 176:
            break
          default:
            ElMessage.error(body.message)
            data.waitLogin = false
            clearInterval(data.timer)
            break
        }
      } catch (error) {
        clearInterval(data.timer)
        data.waitLogin = false
      }
    }

    const CKLogin = async () => {
      const ptKey =
        data.cookie.match(/pt_key=(.*?);/) &&
        data.cookie.match(/pt_key=(.*?);/)[1]
      const ptPin =
        data.cookie.match(/pt_pin=(.*?);/) &&
        data.cookie.match(/pt_pin=(.*?);/)[1]
      if (ptKey && ptPin) {
        const body = await CKLoginAPI({ pt_key: ptKey, pt_pin: ptPin })
        if (body.data.eid) {
          localStorage.setItem('eid', body.data.eid)
          ElMessage.success(body.message)
          router.push('/')
        } else {
          ElMessage.error(body.message || 'cookie 解析失败，请检查后重试！')
        }
      } else {
        ElMessage.error('cookie 解析失败，请检查后重试！')
      }
    }

    // 新增 wskey登录
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
          router.push('/')
        } else {
          ElMessage.error(body.message || 'wskey 解析失败，请检查后重试！')
        }
      } else {
        ElMessage.error('wskey 解析失败，请检查后重试！')
      }
    }

    onMounted(() => {
      getInfo()
      getQrcode()
    })

    return {
      ...toRefs(data),
      getInfo,
      getQrcode,
      showQrcode,
      ckeckLogin,
      jumpLogin,
      CKLogin,
      WSCKLogin,
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
