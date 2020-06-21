<template>
  <div>
    <div>导航</div>
    <nuxt></nuxt>
    <div>底部导航</div>
  </div>
</template>

<script>
    export default {
        name: "login"
    }
</script>

<style lang="stylus">
  .login-container
    width 100%
    min-height 100%
    .login-form
      width 520px
      padding 160px 0
      margin 0 auto
      overflow hidden
    .title-container
      text-align center
      margin-bottom 20px
      img
        height 50px
    .captcha-container
      width 340px
      position relative
      .captcha
        position absolute
        right -110px
        img
          width 90px
          height 50px
          cursor pointer
</style>
