<template>
  <div>
    <v-app-bar app height="60px">
      <v-btn @click="colorPicker = !colorPicker" color="primary">
        <span class="mr-2">Color</span>
        <v-icon>mdi-eyedropper</v-icon>
      </v-btn>
      <v-slider v-model="strokeWidth" min="1" max="10" :thumb-size="24" thumb-label="always" label="stroke" style="margin:45px 10px 0;"></v-slider>

      <v-btn
        @click="
          playSound();
          serverSend({ type: 11 });
        "
        color="primary"
        style="margin-left:10px;"
      >
        <v-icon>mdi-thumb-up</v-icon>
      </v-btn>

      <v-spacer></v-spacer>
      <v-btn
        @click="
          clearLayer({ userId: user.id });
          serverSend({ type: 0 });
        "
        color="primary"
      >
        <span class="mr-2">Clear</span>
        <v-icon>mdi-delete</v-icon>
      </v-btn>
    </v-app-bar>

    <!-- canvas -->
    <v-stage ref="stage" :config="configKonva"> </v-stage>

    <!-- 2 -->
    <div style="position:absolute;top:0px;width:100%;padding:10px;">
      <v-avatar @click="switchLayer(user.id, $event)" v-for="user in users" v-bind:key="user.id" style="margin-left:10px;">
        <img :src="user.pictureUrl" />
      </v-avatar>

      <v-color-picker v-if="colorPicker" @input="colorPicker = !colorPicker" v-model="penColor" hide-inputs style="margin: 20px;"></v-color-picker>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';
import Konva from 'konva';
const clapSound = new Audio('clapping.m4a');
const wwsHost = 'https://foursheets.work/';
// const wwsHost = 'https://webdraw.glitch.me/';

export default {
  name: 'Draw',
  data() {
    return {
      roomName: '',
      user: { id: '', name: '', pictureUrl: '' },
      users: [],
      penColor: '#ff0000',
      colorPicker: false,
      strokeWidth: 4,
      socket: null,
      isPaint: false,
      mode: 'brush',
      configKonva: {
        width: window.innerWidth,
        height: window.innerHeight - 25,
        opacity: 1,
      },
      stage: null,
      layers: [],
      lines: [],
    };
  },
  created() {
    this.socket = io(wwsHost + 'nsp_draw');
    this.roomName = this.$route.params.roomName;
    this.socket.emit('join room', this.roomName);
  },
  mounted() {
    this.stage = this.$refs.stage.getStage();

    window.setProfile = (userId, displayName, pictureUrl) => {
      this.user.id = userId;
      this.user.name = displayName;
      this.user.pictureUrl = pictureUrl;
      this.addUser(this.user);
      this.socket.emit('add user', { user: this.user, room: this.roomName });
    };

    const ua = navigator.userAgent;
    if (ua.indexOf('iPhone') > 0 || ua.indexOf('iPad') > 0 || (ua.indexOf('Android') > 0 && ua.indexOf('Mobile') > 0)) {
      console.log('in ios or android app');
    } else {
      // test for pc browser
      window.setProfile('itamoto' + Math.floor(Math.random() * 100), '板本', 'https://i.pravatar.cc/64');
    }

    this.stage.on('mousedown touchstart', () => {
      const pos = this.stage.getPointerPosition();
      this.newLine({ x: pos.x, y: pos.y, color: this.penColor, strokeWidth: this.strokeWidth, userId: this.user.id });
      this.serverSend({ x: pos.x, y: pos.y, color: this.penColor, strokeWidth: this.strokeWidth, type: 1 });
    });

    this.stage.on('mouseup touchend', () => {
      this.isPaint = false;
    });

    // and core function - drawing
    this.stage.on('mousemove touchmove', () => {
      if (!this.isPaint) {
        return;
      }
      const pos = this.stage.getPointerPosition();
      this.drawLine({ x: pos.x, y: pos.y, userId: this.user.id });

      this.serverSend({ x: pos.x, y: pos.y, color: this.penColor, type: 2 });
    });

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
      if (!this.layers[userData.id]) {
        this.users.push(userData);
        this.layers[userData.id] = new Konva.Layer();
        this.stage.add(this.layers[userData.id]);

        // 新しくユーザが参加した時に自分のLayerを作ってもらう為に送信
        this.socket.emit('add user', { user: this.user, room: this.roomName });
      }
    },
    newLine({ x: x, y: y, color: color, strokeWidth: strokeWidth, userId: userId }) {
      if (this.layers[userId].alpha() == 0) {
        return;
      }
      this.isPaint = true;
      const line = new Konva.Line({
        stroke: color,
        strokeWidth: strokeWidth,
        globalCompositeOperation: this.mode === 'brush' ? 'source-over' : 'destination-out',
        points: [x, y],
      });
      this.lines[userId] = line;
      this.layers[userId].add(this.lines[userId]);
    },

    drawLine({ x: x, y: y, userId: userId }) {
      if (this.layers[userId].alpha() == 0) {
        return;
      }
      const line = this.lines[userId];
      if (line) {
        const newPoints = line.points().concat([x, y]);
        line.points(newPoints);

        this.layers[userId].batchDraw();
      }
    },
    switchLayer(userId, e) {
      console.log(e.currentTarget.style.backgroundColor);
      if (this.layers[userId]) {
        if (this.layers[userId].alpha() == 1) {
          this.layers[userId].clear();
          this.layers[userId].opacity(0);
          this.layers[userId].alpha(0);
        } else {
          this.layers[userId].clear();
          this.layers[userId].opacity(1);
          this.layers[userId].alpha(1);
          this.layers[userId].batchDraw();
        }
      }
    },
    clearLayer({ userId: userId }) {
      if (this.layers[userId]) {
        this.layers[userId].clear();
        this.layers[userId].destroyChildren();
      }
    },
    playSound() {
      clapSound.play();
    },
    serverSend({ x, y, color, type }) {
      // サーバへメッセージ送信
      this.socket.emit('server send', { data: { x: x, y: y, color: color, type: type, userId: this.user.id }, room: this.roomName });
    },
    serverRecieve() {
      this.socket.on('send user', (msg) => {
        if (msg.type == 1) {
          this.newLine({ x: msg.x, y: msg.y, color: msg.color, strokeWidth: msg.strokeWidth, userId: msg.userId });
        } else if (msg.type == 2) {
          this.drawLine({ x: msg.x, y: msg.y, userId: msg.userId });
        } else if (msg.type == 0) {
          this.clearLayer({ userId: msg.userId });
        } else if (msg.type == 11) {
          this.playSound();
        }
      });
      this.socket.on('add user', (msg) => {
        this.addUser(msg);
      });
      // 自分のいるroom以外のuserのdisconnectも受信してしまう
      this.socket.on('user disconnected', (user) => {
        this.users = this.users.filter((e) => e.id != user.id);
        if (this.layers[user.id]) {
          this.layers[user.id].remove();
        }
      });
    },
  },
};
</script>

<style scoped lang="scss"></style>
