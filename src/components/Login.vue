<script setup>
import { inject, ref } from "vue"
import { useRouter } from "vue-router"
import io from "socket.io-client"

// #region global state
const userName = inject("userName")
const alert_username = ref();
alert_username.value = false;
// #endregion

// #region local variable
const router = useRouter()
const socket = io()
// #endregion

// #region reactive variable
const inputUserName = ref("")
// #endregion

// #region browser event handler
// 入室メッセージをクライアントに送信する
const onEnter = () => {
  // ユーザー名が入力されているかチェック
  if (userName.value == "") {
    alert_username.value = true;
    setTimeout(() => {alert_username.value = false}, 3000);
    //alert("ユーザー名を入力してください");
    return false;
  } else {
  // 入室メッセージを送信
  socket.emit("enterEvent", userName.value)
  }
  
  // 全体で使用するnameに入力されたユーザー名を格納

  // チャット画面へ遷移
  router.push({ name: "chat" })
}
// #endregion

const onKeydownEnter = (e) => {
  if (e.keyCode == 229) return
  onEnter();
};
</script>

<template>
  <div class="mx-auto my-5 px-4">
    <v-container class="grey lighten-5">
      <v-alert border="top" color="red" elevation="15" type="error" transition="slide-y-transition" v-model="alert_username">ユーザ名を入力してください</v-alert>
      <h1>ログイン画面</h1>
      <div class="mt-10">
        <v-text-field label="ユーザ名" v-model="userName" @keydown.enter.exact="onKeydownEnter"></v-text-field>
      </div>
      <v-btn block color="accent" elevation="2" @click="onEnter">入室する</v-btn>
    </v-container>
  </div>
</template>

<style scoped>
.user-name-text {
  width: 200px;
  border: 1px solid #888;
  margin-bottom: 16px;
}
</style>