<template>
  <el-row class="outline">
    <el-col class="top">
      <el-col>
        <div v-if="selectedChatRoom.dealStatus === 'BOOK'" class="image-cover">
          <p>예약중</p>
        </div>
        <div v-else-if="selectedChatRoom.dealStatus === 'FINISH'" class="image-cover2">
          <pre>거래 완료</pre>
        </div>
        <el-avatar shape="square" :size="60" fit="scale-down" :src="selectedChatRoom.postImagePath">
        </el-avatar>
        <span class="title">{{ selectedChatRoom.postTitle }}</span>
      </el-col>
      <el-col class="button">
        <el-dropdown>
          <el-button style="margin-right: 10px">거래 상태 변경</el-button>
          <el-dropdown-menu slot="dropdown">
            <el-dropdown-item>
              <div @click="book(selectedChatRoom.postId)">예약</div>
            </el-dropdown-item>
            <el-dropdown-item divided>
              <div @click="cancel(selectedChatRoom.postId)">예약 취소</div>
            </el-dropdown-item>
            <el-dropdown-item divided>
              <div @click="finish(selectedChatRoom.postId)">거래완료</div>
            </el-dropdown-item>
          </el-dropdown-menu>
        </el-dropdown>
        <el-button>채팅 삭제</el-button>
      </el-col>
    </el-col>
    <el-col class="mid" v-chat-scroll>
      <el-row class="one-block" v-for="message in $store.state.message"
              :key="message.chatRoomId">
        <div class="owner" v-if="$store.state.auth.loginInfo.nickname === message.senderNickname">
          <el-avatar :size="50" :src="message.senderProfileImagePath" style="margin-left: 13px" alt="작성자프로필"/>
          <pre class="message">{{ message.message }}</pre>
          <p class="date">{{ message.createDate.substring(11, 16) }}</p>
        </div>
        <div v-else class="receiver">
          <el-avatar :size="50" :src="message.senderProfileImagePath" style="margin-right: 13px" alt="작성자프로필"/>
          <pre class="message2">{{ message.message }}</pre>
          <p class="date">{{ message.createDate.substring(11, 16) }}</p>
        </div>
      </el-row>
    </el-col>


    <el-col class="bottom">
      <el-input class="text"
                v-model="message"
                type="textarea"
                resize="none"
                :row="2" placeholder="보낼 메시지를 작성하세요 ..."/>
      <el-button class="send-button" @click="send" icon="el-icon-s-promotion"/>
    </el-col>
  </el-row>
</template>

<script>
import Stomp from 'webstomp-client'
import SockJS from 'sockjs-client'

export default {
  name: "Message",
  props: {
    selectedChatRoom: {
      type: Object,
      default: () => {
      }
    }
  },
  data() {
    return {
      username: '',
      message: '',
      recvList: [],
    }
  },
  created() {
    this.connect()
    this.$store.dispatch('fetchMessage', this.selectedChatRoom.chatRoomId)
  },
  beforeDestroy() {
    this.stompClient.disconnect();
  },
  methods: {
    async send() {
      if (this.stompClient && this.stompClient.connected) {
        const msg = {
          chatRoomId: this.selectedChatRoom.chatRoomId,
          senderId: this.$store.state.auth.loginInfo.accountId,
          message: this.message
        };
        this.stompClient.send(`/pub/chat/${this.selectedChatRoom.chatRoomId}`, JSON.stringify(msg), {});
        this.message = ""
        await this.$store.dispatch('fetchMessage', this.selectedChatRoom.chatRoomId)
      }
    },
    // eslint-disable-next-line require-await
    async connect() {
      const serverUrl = "http://192.168.10.122:9090/chats"
      const socket = new SockJS(serverUrl);
      this.stompClient = Stomp.over(socket);


      this.stompClient.connect(
        {},
        frame => {
          this.connected = true;
          this.stompClient.subscribe(`/sub/${this.selectedChatRoom.chatRoomId}`, async res => {
            console.log('구독으로 받은 메시지 입니다.', res.body);
            await this.$store.dispatch('fetchMessage', this.selectedChatRoom.chatRoomId)
            this.recvList.push(JSON.parse(res.body))
          });
        },
        error => {
          console.log('소켓 연결 실패', error);
          this.connected = false;
        }
      );
    },
    book(id) {
      this.$confirm('게시물을 예약하시겠습니까?')
        .then(async _ => {
          await this.$axios.put(`/api/posts/${id}/book`, {},
            {
              headers: {
                Authorization: 'Bearer ' + this.$store.state.auth.loginInfo.accessToken
              }
            });
          this.$message({
            message: '예약이 완료되었습니다.',
            type: 'success',
          })
          this.$emit('refresh', 'BOOK')

        })
    },
    async cancel(id) {
          await this.$axios.put(`/api/posts/${id}/yet`, {},
            {
              headers: {
                Authorization: 'Bearer ' + this.$store.state.auth.loginInfo.accessToken
              }
            });
          this.$message({
            message: '예약이 취소되었습니다.',
            type: 'warning',
          })
          this.$emit('refresh', 'YET')

    },
    async finish(id) {
      await this.$axios.put(`/api/posts/${id}/finish`, {},
        {
          headers: {
            Authorization: 'Bearer ' + this.$store.state.auth.loginInfo.accessToken
          }
        });
      this.$emit('refresh', 'FINISH')
    }
  },

}
</script>

<style scoped>
.outline {
  display: flex;
  flex-direction: column;
  height: 68vh;
  padding: 20px;
}

.image-cover {
  position: absolute;
  display: flex;
  height: 60px;
  width: 60px;
  align-items: center;
  justify-content: center;
  color: #BFE4E0;
  background-color: rgba(55, 55, 55, 0.7);
  border-radius: 5px;
  z-index: 100;
}

.image-cover2 {
  position: absolute;
  display: flex;
  height: 60px;
  width: 60px;
  align-items: center;
  justify-content: center;
  color: lightcoral;
  background-color: rgba(55, 55, 55, 0.7);
  border-radius: 5px;
  z-index: 100;
}


.top {
  display: flex;
  background-color: #EDF0F4;
  align-items: center;
  border-bottom: #E5E7EC solid;
  border-bottom-width: 2px;
  padding: 20px;
  border-radius: 20px 20px 0 0;
}

.mid {
  background-color: #EDF0F4;
  height: 100%;
  overflow-y: auto;
}

.bottom {
  display: flex;
  background-color: #EDF0F4;
  justify-content: center;
  padding: 20px;
  height: 110px;
  border-radius: 0 0 20px 20px;
}

.text {
  width: 100%;
  justify-content: center;
  /*box-shadow: 0 1px 80px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04);*/

}

.send-button {
  position: absolute;
  right: 50px;
  bottom: 47.5px;
  background-color: #18A48A;
  color: white;
  width: 55px;
  height: 55px;
  border-radius: 15px;
}


.button {
  display: flex;
  justify-content: flex-end;
}

.title {
  font-size: 18px;
  font-family: "Noto Sans KR", sans-serif;
}

.one-block {
  display: flex;
  flex-direction: column;
  padding: 10px 20px;
}

.owner {
  display: flex;
  flex-direction: row-reverse;
}

.receiver {
  display: flex;
  /*flex-direction: row;*/
}

.message {
  background-color: #BFE4E0;
  padding: 10px;
  border-radius: 10px;
  color: #344B46;
  font-family: "Noto Sans KR", sans-serif;
  word-break: break-all;
  white-space: pre-wrap;
  max-width: 500px;
}

.message2 {
  background-color: white;
  font-family: "Noto Sans KR", sans-serif;
  padding: 10px;
  border-radius: 10px;
  color: #344B46;
}

.date {
  display: flex;
  align-items: flex-end;
  padding: 0 5px;
}
</style>
