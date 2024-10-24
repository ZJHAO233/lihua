<template>
  <a-flex class="login-background" justify="center" align="center">
<!--      主题切换开关-->
    <head-theme-switch class="head-theme-switch"/>
    <a-flex class="login-content" justify="space-around" align="center" v-if="showContent">
<!--        左侧标题-->
      <div class="slogan" v-if="showContent">
        <a-typography-title class="slogan-title">狸花猫后台管理系统</a-typography-title>
        <a-typography-title :level="2" class="slogan-subhead">
          基于SpringBoot 3.X 和 vue3.X 的
          后台管理系统
        </a-typography-title>
      </div>
      <div class="form">
        <transition name="card" mode="out-in">
<!--      右侧表单-->
          <a-card class="login-card" v-if="showLogin" v-show="show">
            <transition name="form" mode="out-in">
              <div style="margin: 16px" v-if="show">
                <div class="login-title">
                  <a-typography-title :level="2">欢迎登陆狸花猫</a-typography-title>
                  <a-typography-text v-if="errorMessage" type="danger">{{errorMessage}}</a-typography-text>
<!--                    根据配置显示注册-->
                  <div v-if="enableRegister">
                    <a-typography-text>没有账户？</a-typography-text>
                    <a-typography-link @click="showLogin = false">快速注册
                      <RightOutlined/>
                    </a-typography-link>
                  </div>
                </div>
                <a-form :model="loginForm" @finish="handleFinish" :rules="loginRoles">
                  <a-form-item name="username" hasFeedback>
                    <a-input class="login-form-item"
                             autocomplete="off"
                             v-model:value="loginForm.username"
                             placeholder="用户名"
                    >
                      <template #prefix>
                        <UserOutlined class="input-prefix-icon-color"/>
                      </template>
                    </a-input>
                  </a-form-item>
                  <a-form-item name="password" hasFeedback>
                    <a-input-password class="login-form-item"
                                      v-model:value="loginForm.password"
                                      placeholder="密码"
                    >
                      <template #prefix>
                        <LockOutlined class="input-prefix-icon-color"/>
                      </template>
                    </a-input-password>
                  </a-form-item>
                  <a-form-item>
                    <a-flex justify="space-between">
                      <a-checkbox v-model:checked="rememberMe">记住账号</a-checkbox>
                      <a-typography-link v-if="false">忘记密码</a-typography-link>
                    </a-flex>
                  </a-form-item>
                  <a-form-item>
                    <a-button html-type="submit"
                              type="primary"
                              class="login-form-item"
                              :loading="loginLoading"
                              style="width: 100%">登录
                    </a-button>
                  </a-form-item>
                </a-form>
                <a-divider plain v-if="false">其他方式</a-divider>
                <a-flex justify="space-around" v-if="false">
                  <a-button size="large" shape="circle"><WeiboCircleOutlined /></a-button>
                  <a-button size="large" shape="circle"><WechatOutlined /></a-button>
                  <a-button size="large" shape="circle"><AlipayCircleOutlined /></a-button>
                  <a-button size="large" shape="circle"><QqOutlined /></a-button>
                  <a-button size="large" shape="circle"><GitlabOutlined /></a-button>
                </a-flex>
              </div>
            </transition>
          </a-card>
<!--            用户注册-->
          <user-register v-else @go-login="handleShowLogin"/>
        </transition>
      </div>
    </a-flex>
    <!--    验证码-->
    <transition name="verify" mode="out-in">
      <verify
          v-if="enableCaptcha"
          @success="login"
          @error="loadVerify"
          ref="verifyRef"
          :captcha-type="captchaType"
          :imgSize="{ width: '330px', height: '155px' }"
      />
    </transition>
    <!--    登录设置-->
    <transition name="setting" mode="out-in">
      <login-setting :component-names="settingComponentNames"
                     v-if="showSetting"
                     @go-login="handleGoLogin"
      ></login-setting>
    </transition>
  </a-flex>

</template>

<script setup lang="ts">
import {useUserStore} from "@/stores/modules/user"
import {useSettingStore} from "@/stores/modules/setting.ts";
import {useRouter} from 'vue-router'
import {onMounted, reactive, ref, useTemplateRef} from "vue"
import {message} from "ant-design-vue";
import Verify from "@/components/verifition/index.vue";
import type {Rule} from "ant-design-vue/es/form";
import {enable} from "@/api/system/captcha/Captcha.ts";
import token from "@/utils/Token.ts"
import HeadThemeSwitch from "@/components/light-dark-switch/index.vue"
import LoginSetting from "@/components/login-setting/index.vue"
import UserRegister from "@/components/user-register/index.vue"
import {ResponseError, type ResponseType} from "@/api/global/Type.ts";
import type {SignIn} from "@/api/system/setting/type/SignIn.ts";
import {init} from "@/utils/AppInit.ts";
import Token from "@/utils/Token.ts";

const router = useRouter()
const verifyRef = useTemplateRef<InstanceType<typeof Verify>>("verifyRef")
const rememberMe = ref<boolean>(token.enableRememberMe())
const showSetting = ref<boolean>(false)
const showContent = ref<boolean>(true)
const settingComponentNames = ref<string[]>([])
const settingStore = useSettingStore()
const userStore = useUserStore()
// 用户登录
interface LoginFormType {
  username: string,
  password: string
}
// 显示登录卡片
const show = ref<boolean>(false)
// 显示登录/注册组件
const showLogin = ref<boolean>(true)

const loginForm = reactive<LoginFormType>({
  username: '',
  password: ''
})
const loginLoading = ref<boolean>()
const errorMessage = ref<string>()

const loginRoles: Record<string, Rule[]> = {
  username: [{required: true, message: '请输入账号', trigger: 'change'}],
  password: [{required: true, message: '请输入密码', trigger: 'change'}]
}

// 启用记住账号后赋值账号密码
const initLogin = () => {
  if (rememberMe.value) {
    const usernamePassword = token.getUsernamePassword()
    loginForm.username = usernamePassword.username
    loginForm.password = usernamePassword.password
  }
}

// 登录请求
const login = async ({captchaVerification}: { captchaVerification: string }) => {
  loginLoading.value = true
  try {
    const {code, msg}: ResponseType<string> = await userStore.login(loginForm.username, loginForm.password, captchaVerification);
    if (code === 200) {
      if (rememberMe.value) {
        token.rememberMe(loginForm.username, loginForm.password)
      } else {
        token.forgetMe()
      }

      // 检查是否需要登录后设置
      const loginSettingResp = await userStore.checkLoginSetting()
      if (loginSettingResp.code === 200) {
        // 登录后设置返回data为空，表示无需进行额外设置，进入首页
        if (loginSettingResp.data === null) {
          message.success("登录成功")
          await router.push("/index");
        } else {
          hideVerify()
          await init()
          // 加载登录配置
          showSetting.value = true
          showContent.value = false
          settingComponentNames.value.push(...loginSettingResp.data.split(","))
          // 关闭登录卡片
          show.value = false
        }
      } else {
        message.error(loginSettingResp.msg)
      }
    } else {
      message.error(msg);
      loginLoading.value = false
    }
  } catch (e) {
    if (e instanceof ResponseError) {
      message.error(e.msg);
    } else {
      console.error("登录失败:", e);
    }
  } finally {
    Token.removeLoginSettingResult()
    loginLoading.value = false
  }
}

// 触发登录
const handleFinish = () => {
  if (enableCaptcha.value) {
    showVerify()
  } else {
    login({captchaVerification: 'loginCaptcha'})
  }
}

// 验证码
const enableCaptcha = ref<boolean>(false)
const captchaType = ref<string>('blockPuzzle')
// 随机加载滑块/点击验证码
const loadVerify = () => {
  setTimeout(() => {
    captchaType.value = Math.random() < 0.5 ? 'blockPuzzle' : 'clickWord';
  },700)

}
// 显示验证码
const showVerify = () => {
  verifyRef.value.show()
}
// 关闭验证码
const hideVerify = () => {
  verifyRef.value.closeBox()
}
// 是否启用验证码
const captcha = async () => {
  try {
    const resp = await enable()
    if (resp.code === 200) {
      enableCaptcha.value = resp.data
    } else {
      message.error(resp.msg)
    }
  } catch (e) {
    if (e instanceof ResponseError) {
      errorMessage.value = '无法访问服务器，请检查网络连接'
    } else {
      console.error(e)
    }
  }
}

// 判断重定向回来的参数，给用户合适的提示
// 提示信息显示完成后销毁，刷新页面时不再提示
const handleRedirect = () => {
  if (history.state.msg) {
    message.error(history.state.msg)
    history.state.msg = undefined
  }
}

// 当需要登陆后配置时，刷新页面读取路由携带参数，加载配置页面
const routerCheckLoginSetting = () => {
  if (history.state.data) {
    showSetting.value = true
    showContent.value = false
    settingComponentNames.value.push(...history.state.data.split(","))
  }
}

// 动画效果
const transition = () => {
  setTimeout(() => {
    show.value = true
  }, 100)
}

// 从注册页面/配置页面切换到登录页面
const handleShowLogin = (clearLoginForm: boolean) => {
  showLogin.value = true
  show.value = false

  // 根据参数清空表单
  if (clearLoginForm) {
    loginForm.username = ''
    loginForm.password = ''
    rememberMe.value = false
  }

  setTimeout(() => {
    show.value = true
  }, 100)
}

// 从配置页面退回到登录页面
const handleGoLogin = async () => {
  // 调用退出接口
  await userStore.handleLogout()
  // 关闭设置页面
  showSetting.value = false
  settingComponentNames.value = []
  // 清空路由参数
  if (history.state.data) {
    history.state.data = undefined
  }
  // 没有勾选记住账号情况下清空账号密码
  if (!rememberMe.value) {
    loginForm.username = ''
    loginForm.password = ''
  }
  setTimeout(() => {
    showContent.value = true
    // 登录卡片弹出动画
    handleShowLogin(false)
  }, 620)

}

const enableRegister = ref<boolean>(false)
const initRegisterSetting = async () => {
  const resp = await settingStore.getSetting<SignIn>('SignInSetting')
  if (resp) {
    enableRegister.value = resp.enable || false
  }
}

onMounted(() => {
  routerCheckLoginSetting()
  transition()
  initLogin()
  handleRedirect()
  captcha()
  initRegisterSetting()
})

</script>

<style scoped>
.login-background {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background-image:linear-gradient(-135deg,#C2FFD8 10%,#465EFB 100%);
  background-size: 200% 200%;
  animation: gradientAnimation 30s ease infinite;
}

.slogan-title {
  margin-top: 64px;
  margin-right: 64px;
  margin-left: 64px
}

.slogan-subhead {
  margin-right: 64px;
  margin-left: 64px
}

@keyframes gradientAnimation {
  0% {
    background-position: 0 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0 50%;
  }
}

@media screen and (max-width: 1000px) {
  .slogan {
    transition: all 0.8s ease-in-out;
    display: none;
    width: 0
  }
}

.form {
  width: 50%;
  min-width: 488px
}

.head-theme-switch {
  position: fixed;
  top: 16px;
  right: 24px;
}

.login-content {
  max-width: 1180px;
  width: 85vw;
  height: 70vh;
}

.login-card {
  width: 380px;
  margin: 64px;
  border-radius: 24px;
}

.login-title {
  margin-top: 32px;
  margin-bottom: 64px;
}

.login-form-item {
  height: 48px
}


.card-enter-active {
  transition: all 0.8s ease-in-out;
}

.card-enter-from {
  transform: translateY(80px) scale(88%) rotateY(24deg);
  opacity: 0;
}

.form-enter-active {
  transition: all 0.6s ease-in-out;
}

.form-enter-from {
  transform: translateY(24px);
  opacity: 0;
}


.setting-enter-active {
  transition: all 0.75s ease-in-out;
}
.setting-enter-from {
  transform: translateY(1000px); /* 从下方 100px 开始 */
  opacity: 0; /* 初始透明度为 0，不可见 */
}
.setting-enter-to {
  transform: translateY(0); /* 移动到原始位置 */
  opacity: 1; /* 最终透明度为 1，完全可见 */
}

.setting-leave-active {
  transition: all 0.6s ease-out;
}
.setting-leave-from {
  transform: translateY(0); /* 从原始位置开始 */
  opacity: 1; /* 初始透明度为 1 */
}
.setting-leave-to {
  transform: translateY(1100px); /* 离开时向下移动 100px */
  opacity: 0; /* 最终透明度为 0，完全消失 */
}

</style>

<style>
[data-theme = dark] {
  .login-background {
    background-image: linear-gradient(-135deg, #1F7A56 0%, #2C3690 100%);
  }
}
</style>
