<template>
  <div>
    <Login v-if="!isLogined" @logined="isLogined = true"></Login>
    <Dashboard v-else />
  </div>
</template>

<script>
import Dashboard from "./components/Dashboard.vue";
import Login from "./components/Login.vue";
import moment from "moment";

export default {
  name: "App",
  components: {
    Dashboard,
    Login,
  },
  created() {
    const infoStr = window.localStorage.getItem("userInfo");
    const userInfo = JSON.parse(infoStr);
    if (!infoStr || !userInfo.lastLogin) {
      this.isLogined = false;
      this.$message({
        message: "登录已过期",
        type: "warning",
      });
      return;
    }
    const diff = moment().diff(moment(userInfo.lastLogin), "day");
    console.log(diff);
    if (diff < 3) {
      window.userInfo = userInfo;
      this.isLogined = true;
    }
  },
  data() {
    return {
      isLogined: false,
    };
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 16px;
}
</style>
