<template>
  <div class="login-container">
    <el-form class="login-form" label-width="100px" :model="form" :rules="rules" ref="loginForm">
      <el-form-item prop="email" label="邮箱">
        <el-input v-model="form.email" placeholder="请输入邮箱"></el-input>
      </el-form-item>
      <el-form-item prop="captcha" label="验证码" class="captcha-container">
        <div class="captcha">
          <img :src="code.captcha" alt="" @click="resetCaptcha">
        </div>
        <el-input v-model="form.captcha" placeholder="请输入验证码"></el-input>
      </el-form-item>
      <el-form-item prop="emailcode" label="验证码" class="captcha-container">
        <div class="captcha">
          <el-button @click="sendEmailCode" :disabled="send.timer > 0" type="primary">{{sendText}}</el-button>
        </div>
        <el-input v-model="form.emailcode" placeholder="请输入验证码"></el-input>
      </el-form-item>
      <el-form-item prop="password" label="密码">
        <el-input v-model="form.password" placeholder="请输入密码"></el-input>
      </el-form-item>
      <el-form-item prop="" label="">
        <el-button type="primary" @click.native.prevent="handleLogin">登录</el-button>
      </el-form-item>
    </el-form>
  </div>
</template>

<script>
    import md5 from 'md5'
    export default {
        layout: "login",
        data() {
          return {
            send: {
              timer: 0
            },
            form: {
              email: "1312943214@qq.com",
              password: "123456qq",
              captcha: ""
            },
            rules:{
              email: [
                {required: true, message: "请输入邮箱"},
                {type: "email", message: "请输入正确的邮箱格式"}
              ],
              password: [
                {required: true, pattern:/^[\w_-]{6,12}$/g, message: "请输入6~12位密码"},
              ],
              captcha: [
                {required: true, message: "请输入验证码"},
              ],
              emailcode: [
                {required: true, message: "请输入邮箱验证码"},
              ],
            },
            code:{
              captcha: "/api/captcha"
            }
          }
        },
        methods: {
          handleLogin(){
            this.$refs.loginForm.validate( async valid => {
              if(valid){
                // @todo 发送注册请求
                let obj = {
                  email: this.form.email,
                  password: md5(this.form.password),
                  captcha: this.form.captcha,
                  emailcode: this.form.emailcode
                }
                let ret = await this.$http.post('/user/login', obj)
                console.log(ret)
                if(ret.code == 0){
                  // token的存储 登陆成功 返回token
                  this.$message.success('登陆成功')
                  localStorage.setItem('token', ret.data.token)
                  setTimeout(() => {
                    this.$router.push('/')
                  },500)
                }else{
                  this.$message.success(ret.message)
                }
              }
            })
          },
          resetCaptcha(){
            this.code.captcha = "/api/captcha?_t" + new Date().getTime()
          },
          async sendEmailCode(){
            await this.$http.get('/sendcode?email=' + this.form.email)
            this.send.timer = 10
            this.timer = setInterval(() => {
              this.send.timer -= 1
              if(this.send.timer === 0) {
                clearInterval(this.timer)
              }
            },1000)
          }
        },
        computed: {
          sendText(){
            if(this.send.timer <= 0){
              return '发送'
            }
            return `${this.send.timer}s后发送`
          }
        }
    }
</script>

<style lang="stylus">
  .login-form
    width 800px
    margin 50px auto
</style>
