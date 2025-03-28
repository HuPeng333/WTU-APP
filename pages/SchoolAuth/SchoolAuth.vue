<template>
  <view class="school-auth">
    <view class="school-auth-status-bar"></view>
    <view class="school-auth-exit">
      <text class="iconfont" @click="back">&#xe612;</text>
    </view>
    <view class="school-auth-title">
      <text class="iconfont">&#xe656;</text>
      <text>登录教务系统</text>
    </view>
    <view class="school-auth-form">
      <animated-input placeholder="用户名" class="school-auth-form-input" v-model="username" v-model:error-msg="userNameErrorMsg"/>
      <animated-input placeholder="密码" type="password" class="school-auth-form-input" v-model="password"  v-model:error-msg="passwordErrorMsg"/>
      <view class="school-auth-form-captcha">
        <animated-input placeholder="验证码" class="school-auth-form-input" v-model="captcha" v-model:error-msg="captchaErrorMsg"/>
        <image :src="captchaBase64"/>
      </view>
      <button class="school-auth-form-btn" hover-class="school-auth-form-btn-hover" @click="tryLogin">登录</button>
    </view>
    <view class="school-auth-info">
      <text>密码是以AES加密的方式上传给服务器,加密密匙是由学校服务器提供,本软件无法破解</text>
      <br/>
      <text>密码会被保存到本地, 以便于下次快速登录</text>
    </view>
    <loading-mask ref="loading"/>
    <tip-dialog ref="dialog"/>
  </view>
</template>

<script>
import { onUnmounted, ref } from 'vue'
import { getCaptcha, init, login } from '../../api/schoolApp/schoolAuth'
import { wtuEncrypt } from '../../hook/aes'
import { SAVE_SCHOOL_LOGIN_INFO } from '../../store/mutations-type'
import { useStore } from 'vuex'
import AnimatedInput from '../../component/AnimatedInput/AnimatedInput'
import LoadingMask from '../../component/LoadingMask/LoadingMask'
import { showToast } from '../../hook/utils/TipUtils'
import TipDialog from '../../component/MyDialog/TipDialog'

export default {
  name: 'SchoolAuth',
  components: { TipDialog, LoadingMask, AnimatedInput },
  setup () {
    const store = useStore()
    const isInitSuccess = ref(false)
    const dialog = ref()

    // loading组件
    const loading = ref()

    const username = ref(store.state.eduSystemUser.username)
    const userNameErrorMsg = ref('')
    const password = ref(store.state.eduSystemUser.password)
    const passwordErrorMsg = ref('')
    const captcha = ref('')
    const captchaErrorMsg = ref('')

    let lt = null
    let encryptSalt = null
    let execution = null

    let interval
    /**
     * 初始化登录请求
     */
    const tryInit = () => {
      isInitSuccess.value = false
      init().then(resp => {
        if (!resp) {
          showToast('自动登录成功')
          store.commit(SAVE_SCHOOL_LOGIN_INFO, {})
          uni.navigateBack()
        }
        // success
        isInitSuccess.value = true
        lt = resp.lt
        encryptSalt = resp.encryptSalt
        execution = resp.execution
        // 获取验证码
        tryGetCaptcha()
      }).catch(e => console.log(e)).finally(() => {
        if (!isInitSuccess.value) {
          uni.showToast({
            title: '连接服务器失败, 请检查你的网络',
            icon: 'none',
            position: 'bottom'
          })
          // 重试
          interval = setTimeout(() => {
            tryInit()
          }, 5000)
        }
      })
    }
    tryInit()

    onUnmounted(() => {
      if (interval) {
        console.log('clear')
        clearTimeout(interval)
      }
    })

    const captchaBase64 = ref('')
    /**
     * 尝试获取验证码
     */
    const tryGetCaptcha = () => {
      if (isInitSuccess.value) {
        getCaptcha().then(resp => {
          captchaBase64.value = 'data:image/gif;base64,' + uni.arrayBufferToBase64(resp)
        })
      }
    }

    /**
     * 检查表单信息格式是否正确
     * @return {boolean} 返回true表示表单信息格式不正确
     */
    const isBadForm = () => {
      let findErr = false
      if (username.value.length <= 8 || username.value.length >= 12) {
        userNameErrorMsg.value = '用户名太短或太长'
        findErr = true
      }
      if (username.value.length === 0) {
        userNameErrorMsg.value = '用户名不能为空'
        findErr = true
      }
      if (password.value.length === 0) {
        passwordErrorMsg.value = '密码不能为空'
        findErr = true
      }
      if (captcha.value.length === 0) {
        captchaErrorMsg.value = '验证码不能为空'
        findErr = true
      }
      return findErr
    }

    const tryLogin = () => {
      if (isBadForm()) {
        return
      }
      if (isInitSuccess.value) {
        loading.value.showLoading()
        const encryptedPassword = wtuEncrypt(password.value, encryptSalt)
        let isLoginSuccess = false
        login(lt, encryptedPassword, captcha.value, username.value, execution).then(resp => {
          const match = resp.match(/<span id="msg" class="auth_error".+</)
          if (match == null) {
            // success
            isLoginSuccess = true
            store.commit(SAVE_SCHOOL_LOGIN_INFO, { username: username.value, password: password.value })
            uni.showToast({
              title: '登录成功',
              icon: 'none',
              position: 'bottom'
            })
            uni.navigateBack()
          } else {
            let errMsg = match[0].replace(/<.+>/, '').replace('<', '')
            dialog.value.showDialog({
              title: '登录失败',
              message: errMsg,
              hideCancel: true,
              type: 'error'
            })
          }
          loading.value.stopLoading()
        }).finally(() => {
          loading.value.stopLoading()
          if (!isLoginSuccess) {
            tryInit()
          }
        })
      }
    }

    const back = () => uni.navigateBack()

    return {
      captchaBase64,
      username,
      password,
      captcha,
      tryLogin,
      back,
      userNameErrorMsg: userNameErrorMsg,
      passwordErrorMsg: passwordErrorMsg,
      captchaErrorMsg: captchaErrorMsg,
      loading,
      dialog
    }
  }
}
</script>

<style lang="scss">
.school-auth-info{
  margin-top: 20rpx;
  font-size: 16rpx;
  color: $uni-text-color-grey;
  text-align: center;
}
.school-auth-form-btn-hover{
  background-color: #a9d2ff !important;
}
.school-auth-form-btn{
  width: 630rpx;
  height: 98rpx;
  border-radius: 49rpx;
  margin-top: 30rpx;
  color: #fff;
  border: none;
  transition: background-color 0.3s;
  background-color: $uni-color-primary;
}
.school-auth-form-captcha{
  display: flex;
  align-items: flex-end;
  > image {
    width: 200rpx;
    height: 100rpx;
  }
}
.school-auth-form-input{
  margin-top: 80rpx;
  border-bottom: solid 1rpx #e8e8e8;
}
.school-auth-form{
  width: 100%;
  padding: 0 62rpx;
  box-sizing: border-box;
}
.school-auth-title{
  > text:first-child{
    margin-right: 10rpx;
  }
  margin-left: 40rpx;
  font-size: 48rpx;
  color: $uni-color-primary;
}
.school-auth-exit{
  margin-top: 10rpx;
  margin-right: 25rpx;
  font-weight: bold;
  text-align: end;
}
.school-auth-status-bar{
  height: var(--status-bar-height);
  width: 100%;
}
.school-auth{
  background-color: white;
  height: 100vh;
  width: 100%;
}
</style>
