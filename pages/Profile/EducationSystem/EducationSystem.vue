<template>
  <view>
    <status-bar/>
    <my-navigator title="教务系统" show-back/>
    <view class="edu-system-header">
      <view class="edu-system-header-title">
        <text v-if="isUsableToken">已登录</text>
        <text v-else-if="!isUsableToken">登录失效</text>
        <text v-else>未登录</text>
      </view>
      <view class="edu-system-header-sub-title-text">
        <text v-if="!isUsableToken">登录获取更多支持</text>
      </view>
      <view class="edu-system-header-btn" v-if="!isUsableToken">
        <button @click="jump('/pages/SchoolAuth/SchoolAuth')">登录</button>
      </view>
      <view v-else-if="userInfo.name && loadInfoStatus === 2" class="edu-system-header-info">
        <text>姓名: {{userInfo.name}}</text>
        <text>学号: {{userInfo.idNumber}}</text>
        <text>入学日期: {{userInfo.enterTime}}</text>
      </view>
      <view class="edu-system-header-info" v-else-if="loadInfoStatus === 1">
        <text>加载个人信息超时, 请稍后再试</text>
      </view>
      <view class="edu-system-header-info" v-else>
        <text>加载个人信息中</text>
      </view>
    </view>
    <view v-if="isUsableToken" class="edu-system-config">
      <options-divider/>
      <options-switch title="当登录过期后自动跳转到登录页面" :model-value="autoRedirectLoginPage" :update-callback="setAutoRedirectLoginPage"/>
      <options-divider/>
    </view>
    <tip-dialog ref="dialog"/>
  </view>
</template>

<script>
import { computed, reactive, ref, watchEffect } from 'vue'
import { useStore } from 'vuex'
import { getUserInfo } from '../../../api/schoolApp/schoolAuth'
import { INVALID_EDU_SYSTEM_TOKEN, SET_EDU_SYSTEM_OPTIONS } from '../../../store/mutations-type'
import { PROXY_SCHOOL_APP_AJAX } from '../../../store/actions-type'
import OptionsDivider from '../../../component/OptionsComponent/OptionsDivider/OptionsDivider'
import OptionsSwitch from '../../../component/OptionsComponent/OptionsSwitch/OptionsSwitch'
import MyNavigator from '../../../component/Navigator/Navigator'
import TipDialog from '../../../component/MyDialog/TipDialog'
import StatusBar from '../../../component/Navigator/StatusBar'

export default {
  name: 'EducationSystem',
  components: { StatusBar, TipDialog, MyNavigator, OptionsSwitch, OptionsDivider },
  setup () {
    const dialog = ref()
    const store = useStore()
    const jump = (url) => {
      uni.navigateTo({ url })
    }

    const isUsableToken = computed(() => store.state.eduSystemUser.isUsableToken)
    const username = store.state.eduSystemUser.username

    let userInfo = reactive({
      idNumber: '',
      name: '',
      enterTime: ''
    })

    // 加载个人信息的目前状态, 0: 加载中或还没登录 1: 加载失败 2: 加载成功
    const loadInfoStatus = ref(0)
    /**
     * 发送ajax请求, 获取用户信息
     */
    const getUserInfoAjax = () => {
      if (isUsableToken.value) {
        // 获取用户信息
        store.dispatch(PROXY_SCHOOL_APP_AJAX, getUserInfo(username)).then(resp => {
          if (resp) {
            // 分离值
            const splitValue = (str) => str.replace('<p class="form-control-static">', '').replace('</p>', '')

            const reg = /<p class="form-control-static">.*<\/p>/g
            const values = resp.match(reg)
            if (values.length === 0) {
              // 请求失败, 登录凭据失效
              store.commit(INVALID_EDU_SYSTEM_TOKEN)
              return
            }

            // 学号
            userInfo.idNumber = splitValue(values[0])
            // 姓名
            userInfo.name = splitValue(values[1])
            // 入学日期
            userInfo.enterTime = splitValue(values[10])
            console.log('success')
            loadInfoStatus.value = 2
          } else {
            loadInfoStatus.value = 1
          }
        }).catch(e => {
          loadInfoStatus.value = 1
          console.log(e)
        })
      }
    }

    // 监听用户登录状态
    watchEffect(() => {
      if (isUsableToken.value) {
        getUserInfoAjax()
      }
    })

    // ----------设置栏-------------

    /**
     * 设置自动登录
     * @param val {boolean} val
     */
    const setAutoRedirectLoginPage = (val) => {
      store.commit(SET_EDU_SYSTEM_OPTIONS, { key: 'autoRedirectLoginPage', value: val })
    }
    return {
      token: computed(() => store.state.eduSystemUser.token),
      isUsableToken: computed(() => store.state.eduSystemUser.isUsableToken),
      jump,
      userInfo,
      loadInfoStatus,
      autoRedirectLoginPage: computed(() => !!store.state.eduSystemUser.config.autoRedirectLoginPage),
      setAutoRedirectLoginPage,
      dialog
    }
  }
}
</script>

<style lang="scss">
.edu-system-header-info{
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 20rpx;
  color: white;
  > text {
    margin-top: 20rpx;
  }
}
.edu-system-header-btn{
  > button {
    margin-top: 50rpx;
    width: 200rpx;
    height: 80rpx;
    border-radius: 50px;
    font-size: $uni-font-size-base;
    color: white;
    line-height: 80rpx;
    background-color: rgb(61, 141, 253);
  }
}
.edu-system-header-sub-title-text{
  font-size: 20rpx;
  text-align: center;
  color: $uni-text-color-grey;
}
.edu-system-header-title{
  font-size: 60rpx;
  text-align: center;
  color: $uni-color-primary;
}
.edu-system-header{
  height: 400rpx;
  padding-top: 20rpx;
  background-image: linear-gradient(to top, #64b3f4, #c2e59c);
}

</style>
