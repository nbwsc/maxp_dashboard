<template>
  <div>
    <AuthingGuard :appId="appId" @login="handleLogin" />
  </div>
</template>
<script>
import { AuthingGuard } from "@authing/vue-ui-components";
import "@authing/vue-ui-components/lib/index.min.css";
export default {
  name: "Login",
  components: {
    AuthingGuard,
  },
  data() {
    return {
      appId: "61caeb601dce97d2c40cc179",
    };
  },
  methods: {
    handleLogin(userInfo) {
      window.userInfo = userInfo;
      console.log(userInfo);
      window.localStorage.setItem('userInfo', JSON.stringify(userInfo));
      this.$emit("logined");
    },
  },
};
</script>
