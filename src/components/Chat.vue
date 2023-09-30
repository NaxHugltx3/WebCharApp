<script setup>
import { inject, ref, reactive, onMounted, nextTick } from "vue"
import io from "socket.io-client"

// #region global state
const userName = inject("userName")
// const userName = "Rakus"
// #endregion

// #region local variable
const socket = io()
// #endregion

// #region reactive variable
const chatContent = ref("")
const chatList = reactive([])
const memoList = reactive([])


const error_no_content = ref("")
const warning_time_limit = ref("")
error_no_content.value = false
warning_time_limit.value = false

var Passsec = 0
var PassageID = null

const TimeCount = ()=>{
  Passsec++;
  //console.log(Passsec);
  if(Passsec == 3){
    TimerStop(PassageID);
  }
}

const TimerStart = () =>{
  PassageID = setInterval(TimeCount, 1000);
  //document.getElementById("postbutton").disabled = true;
  //console.log("TimerStart");
}

const TimerStop = () =>{
  Passsec = 0;
  //console.log("Stop!!");
  clearInterval(PassageID);
  //document.getElementById("postbutton").disabled = false;
}

const RealTime = ()=>{
  var NowTime = new Date()
  var Year = NowTime.getFullYear()
  var Month = NowTime.getMonth()
  var date = NowTime.getDate()
  var hour = NowTime.getHours()
  var min = ("0"+NowTime.getMinutes()).slice(-2)

  return `${Year}/${Month + 1}/${date} ${hour}:${min}`
};

// #endregion

// #region lifecycle
onMounted(() => {
  registerSocketEvent()
})
// #endregion

// #region browser event handler
// 投稿メッセージをサーバに送信する
const onPublish = () => {
  // 投稿内容が空でないかを確認
  //console.log("onPublish");
  if (chatContent.value.trim() !== "") {
      if(0 < Passsec) {
        //alert("間隔をあけて投稿してください");
        warning_time_limit.value = true;
        setInterval(() => {
          warning_time_limit.value = false;
        }, 3000);
      return;
    }
    let msg = { 
      name: userName.value,
      content: chatContent.value,
      time : RealTime(),
    };
    socket.emit("publishEvent", msg);
    // 入力欄を初期化
    chatContent.value = ""
    
    nextTick(() => {
      const timeline = document.getElementById("timeline");
      timeline.scrollTo(0, timeline.scrollHeight);
    });
    //timeline.scrollTo(0, timeline.scrollHeight);
    //console.log(msg)
    TimerStart();
  } else {
    // Display an alert if the message is empty
    //alert("メッセージを入力してください。");
    error_no_content.value = true;
    setInterval(() => {
      error_no_content.value = false;
    }, 3000)
  }
}

// 退室メッセージをサーバに送信する
const onExit = () => {
  socket.emit("exitEvent", userName.value)
}

// メモを画面上に表示する
const onMemo = () => {
  // メモの内容を表示
  let msg = {
    name: "自分",
    content: chatContent.value,
    time : RealTime(),
  };

  memoList.push(msg);
  // 入力欄を初期化
  chatContent.value = ""
  nextTick(() => {
      const timeline = document.getElementById("timeline");
      timeline.scrollTo(0, timeline.scrollHeight);
    });
}
// #endregion

// #region socket event handler
// サーバから受信した入室メッセージ画面上に表示する
const onReceiveEnter = (data) => {
  chatList.push(data);
}

// サーバから受信した退室メッセージを受け取り画面上に表示する
const onReceiveExit = (data) => {
  chatList.push(data);
}

// サーバから受信した投稿メッセージを画面上に表示する
const onReceivePublish = (data) => {
  chatList.push(data);
  nextTick(() => {
      const timeline = document.getElementById("timeline");
      timeline.scrollTo(0, timeline.scrollHeight);
    });
}
// #endregion

// #region local methods
// イベント登録をまとめる
const registerSocketEvent = () => {
  // 入室イベントを受け取ったら実行
  socket.on("enterEvent", (data) => {
    console.log(data)
    let msg = {
      name: "システム",
      content: `${data}さんが入室しました。`,
      time: RealTime(),
    }
    chatList.push(msg)
    nextTick(() => {
      const timeline = document.getElementById("timeline");
      timeline.scrollTo(0, timeline.scrollHeight);
    });
  })

  // 退室イベントを受け取ったら実行
  socket.on("exitEvent", (data) => {
    let msg = {
      name: "システム",
      content: `${data}さんが退室しました。`,
      time : RealTime(),
    }
    chatList.push(msg)
    nextTick(() => {
      const timeline = document.getElementById("timeline");
      timeline.scrollTo(0, timeline.scrollHeight);
    });
  })

  // 投稿イベントを受け取ったら実行
  socket.on("publishEvent", (data) => {
    onReceivePublish(data);
  })
}

// #endregion

const onKeydownEnter = (e) => {
  if (e.keyCode == 229) return;
  onPublish();
}

const onKeyupEnter = (e) => {
  if (0 < Passsec) return;
  chatContent.value = "";
}
</script>

<template>
  <!-- 画面上に固定する「チャットルーム」とログインユーザ表示 -->
  <div>
    <v-alert border="top" color="red" elevation="15" type="error" transition="slide-y-transition" v-model="error_no_content">メッセージを入力してください</v-alert>
    <v-alert border="top" elevation="15" type="warning" transition="slide-y-transition" v-model="warning_time_limit">間隔をあけて投稿してください</v-alert>
    <router-link to="/" class="link">
      <!--<button type="button" class="button-normal button-exit" @click="onExit">退室する</button>-->
      <v-btn rounded type="button" class="button-normal button-exit" color="green-lighten-1" @click="onExit">
        <v-icon>mdi-arrow-bottom-left-thick</v-icon>
      </v-btn>
    </router-link>
    <h1 class="text-h3 font-weight-medium" style="text-align: center;">チャットルーム</h1>
    <p style="text-align: right; margin-top: 10px;">ログインユーザ：{{ userName }}さん</p>
  </div>

  <!-- チャットリスト -->
  <ul class="chat-list" id="timeline">
    <li
      class="item mt-3 mb-3" 
      v-for="(chat, i) in chatList"
      :key="i"
      style="display: flex; flex-direction: column; position: relative;"
    >
      <div>
        <div
          :class="{
            'system-message': chat.name === 'システム',
            'my-post': chat.name === userName && chat.name !== 'システム',
            'other-user-post': chat.name !== userName && chat.name !== 'システム',
          }"
          class="balloon"
        >
          <span v-if= "chat.name !== userName"> <v-icon>mdi-account</v-icon> </span>
          <span v-if="chat.name !== 'システム' && chat.name !== userName">{{ chat.name }}さん  </span>
          <span>{{ chat.content }}</span>
          <span class="timestamp" v-if="chat.name !== 'システム'">{{ chat.time }}</span>
        </div>
      </div>
    </li>
    <!-- チャットリスト内のメモリスト -->
    <li
      class="item mt-4"
      v-for="(msg, i) in memoList"
      :key="i"
    >
      <v-icon color="pink-lighten-1">mdi-lead-pencil</v-icon> {{ msg.content }}
    </li>
  </ul>

  <!-- 画面下に固定する部分（テキスト入力＆投稿＆メモ＆退出） -->
  <div class="fixed-bottom-container"> 
    <div class="input-group">

      <!-- <textarea variant="outlined"
        placeholder="投稿文を入力してください"
        rows="2" class="area"
        type="text"
        v-model="chatContent"
        @keydown.enter.exact="onKeydownEnter">
      </textarea>-->
      <!--
      <v-text-field label="投稿文を入力してください" type="text" v-model="chatContent" clearable height="50" @keydown.enter.exact="onKeydownEnter">
        
        <template v-slot:append-inner>
          <v-btn icon="mdi-send" class="button-normal" type="button" height="35" width="35" color="primary" id="postbutton" @click="onPublish">
          </v-btn>
          <v-btn icon="mdi-lead-pencil" class="button-normal util-ml-8px" type="button" height="35" width="35" color="pink-lighten-1" @click="onMemo">
          </v-btn>
        </template>
      </v-text-field>-->
      <v-textarea
          label="メッセージ"
          hint="Enterで送信"
          rows="2"
          @keydown.enter.exact="onKeydownEnter"
          @keyup.enter.exact="onKeyupEnter"
          v-model="chatContent">
          <template v-slot:append-inner>
            <v-btn icon="mdi-send" class="button-normal" type="button" height="35" width="35" color="primary" id="postbutton" @click="onPublish">
            </v-btn>
            <v-btn icon="mdi-lead-pencil" class="button-normal util-ml-8px" type="button" height="35" width="35" color="pink-lighten-1" @click="onMemo">
            </v-btn>
         </template>
        </v-textarea>

      <!--<button class="button-normal" type="button" id="postbutton" @click="onPublish">投稿</button>-->
      <!--<button class="button-normal util-ml-8px" type="button" @click="onMemo">メモ</button>-->

    </div>
    

  </div>

</template>

<style scoped>
.link {
  text-decoration: none;
}

/* 
.area {
  width: 500px;
  border: 1px solid #000;
  margin-top: 8px;
}
*/
.item {
  display: block;
  white-space: pre-wrap;
}

.balloon {
  display: inline-block;
  padding: 10px;
  border-radius: 10px;
  position: relative;
  margin-bottom: 18px; /* チャット同士の感覚を調整 */
}


.timestamp {
  margin-top: 5px; /* マージンを調整 */
  position: absolute; /* 絶対位置指定 */
  bottom: -20px; /* 下側に配置して少し下に表示 */
  color: gray; /* タイムスタンプのテキストをグレーにする */
  font-size: 12px; /* フォントサイズを調整 */
  white-space: nowrap; /* テキストが折り返されないようにする */
}

.my-post {
  background-color: rgba(255, 255, 0, 0.427);
  border: 2px solid orange;
  text-align: left;
  float: right;
  clear: both; /* clear: right; を clear: both; に修正 */
}

.other-user-post {
  background-color: rgba(195, 222, 233, 0.577);
  border: 2px solid blue;
  text-align: left;
  float: left;
  clear: both; /* clear: left; を clear: both; に修正 */
}

/* "システム" のメッセージのスタイル */
.system-message {
  text-align: center; /* 中央揃えにするスタイル */
  clear: both;
  width: 100%; /* 幅を100%に設定して横一列に表示 */
}


.my-post .timestamp {
  right: 0;
}

.other-user-post .timestamp {
  left: 0;
}

.util-ml-8px {
  margin-left: 8px;
}

.button-exit {
  color: #000;
  margin-top: 8px;
}

/* チャット部分コンテナ */
.chat-list {
  max-height: calc(100vh - 300px); /* ビューポートの高さから適当な余白を引いた高さ */
  overflow-y: auto; /* 縦方向のスクロールを有効にする */
  padding-top: 15px 0; /* 上下に余白を追加 */
}

.memo-list {
  overflow-y: auto;
}

/* チャットコンテナ内のそれぞれの要素 */

.input-group {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 10px; /* 縦に余白を追加 */
  margin-left: 20px; /* 左側の余白を追加 */
  margin-right: 20px; /* 右側の余白を追加 */
}

/* 画面下固定コンテナ */
.fixed-bottom-container{
  position: fixed; /* 固定位置に配置 */
  bottom: 20px; /* 画面下部からの距離を調整 */
  left: 0; /* 左端に配置 */
  right: 0; /* 右端に配置 */
  text-align: center; /* ボタンを中央揃えにする */
}

</style>