<template>
    <p>Socket connected: {{ connected ? 'yes' : 'no' }}</p>
    <p>{{ socketId }}</p>

  <div style="display: flex; justify-content: space-around;">
    <div>
      <h3>Chatbox</h3>
      <form @submit.prevent="sendMessage()">
        <input type="text" placeholder="Message..." v-model="message">
        <button type="submit" :disabled="!connected" style="margin-top: 12px;">Send Message</button>
      </form>

      <div style="margin: 0 auto; height: 300px; width: 500px; border: 1px solid black; overflow-y: scroll;">
        <div v-for="message in messages">
          <div style="border-bottom: 1px solid black">
            <div :style="{ textAlign: (message.userId === socketId ? 'right' : 'left')}">
              <p>
                <small>From: {{message.userId}} {{ message.time.toISOString() }}</small>
                
              </p>
              {{ message.message }}
            </div>
          </div>
        </div>
      </div>
    </div>


    <div>
      <h3>Playground</h3>
      Next in {{ timerCount }} sec
      <div style="width: 400px; height: 400px; border: 1px solid red; margin: 0 auto;">
        <template v-for="row in 20">
          <template v-for="column in 20">
            <div @click="setField(row, column)"
              class="pixel-box"
              :style="{backgroundColor: colorMap[`${row}.${column}`]?.color}"
              :title="`${ row }:${ column } ${ colorMap[`${row}.${column}`]?.userId ?? '' }`">
              
            </div>
          </template>
        </template>
      </div>
      
      <div style="display: flex; justify-content: flex-start;">
        <template v-for="color in ['red', 'blue', 'yellow']">
          <div style="margin: 5px; cursor: pointer; border: 1px solid black; border-radius: 5px; padding: 4px;" @click="changeColor(color)">
            {{ color }}
          </div>
        </template>
      </div>
    </div>
  </div>
</template>


<script  lang="ts">
import { defineComponent} from 'vue'
import { io } from "socket.io-client";

let socket;

interface Message {
  message: string,
  userId : string,
  time: Date,
}

export default defineComponent({
  name: 'App',
  components: {},
  data() {
    return {
      connected: false,
      socketId: '',

      message: '',
      messages: [] as Message[],

      nextFieldPossible: null,

      selectedColor: 'red',
      colorMap: {  },
      timerCount: 0,
    }
  },
  created() {
    socket = io('http://localhost:3000');

    socket.on("connect", () => {
      this.socketId = socket.id;
      this.connected = true;
    });
  },
  mounted() {
    socket.on('newMessage', (message, userId, dateTime) => {
      this.messages.unshift({ message, userId, time: new Date(dateTime)})
    })

    socket.on('setField', (x, y, color, userId) => {
      this.colorMap[`${x}.${y}`] = {color, userId}
    })

    socket.on('nextFieldChangePossible', (datetime) => {
      this.nextFieldPossible = new Date(datetime)
      this.timerCount = Number((this.nextFieldPossible.getTime() - new Date().getTime()) / 1000).toFixed(1);
    })
  },
  methods: {
    sendMessage() {
      if (this.message === undefined || this.message.length === 0) {
        return  
      }

      socket.emit("newMessage", this.message)
      this.message = ''
    },
    setField(row: number, column: number) {
      if (this.nextFieldPossible !== null && (this.nextFieldPossible.getTime() - new Date().getTime()) > 0) {
        return
      }

      socket.emit('changeField', row, column, this.selectedColor)
    },
    changeColor(color: string) {
      this.selectedColor = color
    },
  },
  watch: {
    timerCount: {
      handler(value) {
        if (value > 0) {
          setTimeout(() => {
            const number = Number((this.nextFieldPossible.getTime() - new Date().getTime()) / 1000).toFixed(1);
            this.timerCount = number <= 0 ? 0 : number;
          }, 100);
        }
      },
      immediate: true
    }
  }
})
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.pixel-box {
  height: 20px;
  width: 20px;
  float: left;
  box-sizing: border-box;
}
.pixel-box:hover {
  border: 1px solid black;
}
</style>
