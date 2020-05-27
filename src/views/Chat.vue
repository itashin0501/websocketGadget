<template>
  <div>
    <v-avatar @click="changeToUser(user)" v-for="user in users" v-bind:key="user.id" style="margin-left:10px;">
      <img :src="user.pictureUrl" />
    </v-avatar>

    <v-timeline v-for="(data, key) in displayMessages" :key="key" dense>
      <v-timeline-item large>
        <template v-slot:icon>
          <v-avatar>
            <img :src="data.from.pictureUrl" />
          </v-avatar>
        </template>
        <v-card class="elevation-3 baloon">
          <v-card-text> {{ data.message }} </v-card-text>
        </v-card>
      </v-timeline-item>
    </v-timeline>

    <footer>
      <v-row class="mb-12" align-content="center" justify="center" no-gutters>
        <v-avatar>
          <img :src="toUser.pictureUrl" />
        </v-avatar>
        <v-textarea v-model="message" rows="2" style="margin:0 10px;" />
        <v-btn @click="sendDirectMessage()" color="primary">
          <span class="mr-2">Send</span>
          <v-icon>mdi-send</v-icon>
        </v-btn>
      </v-row>
    </footer>
  </div>
</template>

<script>
import io from 'socket.io-client';
const wwsHost = 'https://foursheets.work/';
// const wwsHost = 'https://webdraw.glitch.me/';

export default {
  name: 'Chat',
  data() {
    return {
      roomName: '',
      user: { id: '', name: '', pictureUrl: '' },
      users: [],
      toUser: { id: '', name: '', pictureUrl: '' },
      socket: null,
      message: '',
      recieveMessages: {},
      displayMessages: [],
    };
  },
  created() {
    this.socket = io(wwsHost + 'nsp_chat');
    this.roomName = this.$route.params.roomName;
    this.socket.emit('join room', this.roomName);
  },
  mounted() {
    window.setProfile = (userId, displayName, pictureUrl) => {
      this.user.id = userId;
      this.user.name = displayName;
      this.user.pictureUrl = pictureUrl;
      this.addUser(this.user);
      this.toUser = this.user;
      this.socket.emit('add user', { user: this.user, room: this.roomName });
      this.socket.emit('get room users', this.roomName);
    };

    const ua = navigator.userAgent;
    if (ua.indexOf('iPhone') > 0 || ua.indexOf('iPad') > 0 || (ua.indexOf('Android') > 0 && ua.indexOf('Mobile') > 0)) {
      console.log('in ios or android app');
    } else {
      // test for pc browser
      window.setProfile('itamoto' + Math.floor(Math.random() * 100), '板本', 'https://i.pravatar.cc/64');
    }

    this.serverRecieve();
  },
  computed: {
    color: {
      get() {
        return this[this.type];
      },
      set(v) {
        this[this.type] = v;
      },
    },
  },
  methods: {
    addUser(userData) {
      this.users.push(userData);
    },
    changeToUser(toUser) {
      this.toUser = toUser;
      if (!this.recieveMessages[toUser.id]) {
        this.recieveMessages[toUser.id] = [];
      }
      this.displayMessages = this.recieveMessages[toUser.id];
    },
    sendDirectMessage() {
      this.socket.emit('direct message', { toUserId: this.toUser.id, message: this.message });
    },
    serverRecieve() {
      this.socket.on('add user', (user) => {
        this.addUser(user);
      });
      this.socket.on('get room users', (users) => {
        users.forEach((user) => {
          if (user.id != this.user.id) {
            this.addUser(user);
          }
        });
      });
      this.socket.on('direct message', (data) => {
        if (!this.recieveMessages[data.from.id]) {
          this.recieveMessages[data.from.id] = [];
        }
        this.recieveMessages[data.from.id].push(data);
        this.displayMessages = this.recieveMessages[data.from.id];
      });

      // 自分のいるroom以外のuserのdisconnectも受信してしまうが、idが存在しないはずなので問題ない。
      this.socket.on('user disconnected', (user) => {
        this.users = this.users.filter((e) => e.id != user.id);
      });
    },
  },
};
</script>

<style scoped lang="scss">
footer {
  width: 100%;
  background-color: #89c7de;
  color: #fff;
  text-align: center;
  padding: 10px;

  position: absolute; /*←絶対位置*/
  bottom: 0; /*下に固定*/
}
</style>
