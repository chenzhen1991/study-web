<template>
    <div class="login-container">
      <el-form class="login-form" label-width="100px" :model="form" :rules="rules" ref="registerForm">
        <div class="title-container">
          <img src="/logo.png" alt="">
        </div>
        <el-form-item props="email" label="邮箱">
          <el-input v-model="form.email" placeholder="请输入邮箱"></el-input>
        </el-form-item>
        <el-form-item props="captcha" label="验证码" class="captcha-container">
          <div class="captcha">
            <img :src="code.captcha" alt="">
          </div>
          <el-input v-model="form.captcha" placeholder="请输入验证码"></el-input>
        </el-form-item>
        <el-form-item props="nickname" label="昵称">
          <el-input v-model="form.nickname" placeholder="请输入昵称"></el-input>
        </el-form-item>
        <el-form-item props="passwd" label="密码">
          <el-input v-model="form.passwd" placeholder="请输入密码"></el-input>
        </el-form-item>
        <el-form-item props="repasswd" label="确认密码">
          <el-input v-model="form.repasswd" placeholder="请再次输入密码"></el-input>
        </el-form-item>
        <el-form-item props="" label="">
          <el-button type="primary" @click.native.prevent="handleRegister">注册</el-button>
        </el-form-item>
      </el-form>
    </div>
</template>

<script>
    export default {
        layout: "login",
        data() {
          return {
            form :{
              email: "",
              nickname: "",
              passwd: "",
              repasswd: "",
              captcha: ""
            },
            rules:{
              email: [
                {require: true, message: "请输入邮箱"},
                {type: "email", message: "请输入正确的邮箱格式"}
              ],
              captcha: [
                {require: true, message: "请输入邮箱"},
              ],
              nickname: [
                {require: true, message: "请输入昵称"},
              ],
              passwd: [
                {require: true, pattern:/^[\w_-]{6,12}$/g, message: "请输入6~12位密码"},
              ],
              repasswd: [
                {require: true, message: "请输入再次输入密码"},
                {validator: (rule, value, callback) => {
                  if(value !== this.form.passwd) {
                    callback(new Error('两次密码不一致'))
                  }
                  callback()
                }}
              ]
            },
            code:{
              captcha: "/api/captcha"
            }
          }
        }
    }
</script>

<style lang="stylus">

</style>
