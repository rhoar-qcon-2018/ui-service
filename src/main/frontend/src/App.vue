<template>
  <div id="q-app">
    <main>
      <img width="450px" height="auto" src="~assets/logo.png" alt="Red Hat OpenShift Application Runtimes - Insult UI" />
    </main>
    <view-space>
      <router-view></router-view>
    </view-space>
  </div>
</template>

<script>
/*
 * Root component
 */
export default {}
</script>

<style lang="stylus">
@import '~variables'

main
  width 450px
  margin auto
  margin-top 40px

view-space
  width 100%

header
  margin 0
  height 10px
  background-color $primary
</style>
