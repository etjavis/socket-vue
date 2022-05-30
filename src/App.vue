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
      Next in {{ timerCount.toFixed(1) }} sec
      <div style="width: 400px; height: 400px; border: 1px solid red; margin: 0 auto;">
        <template v-for="row in 20">
          <template v-for="column in 20">
            <div @click="setField(row, column)"
              class="pixel-box"
              :style="{backgroundColor: colorMap.get(`${row}.${column}`)?.color}"
              :title="`${ row }:${ column } ${ colorMap.get(`${row}.${column}`)?.userId ?? '' }`">
              
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


<script setup lang="ts">
import { io, Socket } from "socket.io-client";
import { onMounted, onBeforeMount, ref, watch } from "vue";

interface Message {
  message: string,
  userId : string,
  time: Date,
}
interface PixelInformation {
  color: string,
  userId: string
}

let socket: Socket;

const connected = ref<Boolean>(false)
const socketId = ref('')

const message = ref('')
const messages = ref<Message[]>([])

const nextFieldPossible = ref<Date>(new Date())

const selectedColor = ref('red')
const timerCount = ref<Number>(0)
const colorMap = ref<Map<string, PixelInformation>>(new Map())

onBeforeMount(() => {
  socket = io('http://localhost:3000');

  socket.on("connect", () => {
    socketId.value = socket.id;
    connected.value = true;
  });
})

onMounted(() => {
  socket.on('newMessage', (message, userId, dateTime) => {
    messages.value.unshift({ message, userId, time: new Date(dateTime)})
  })

  socket.on('setField', (x, y, color, userId) => {
    colorMap.value.set(`${x}.${y}`, {color, userId})
  })

  socket.on('nextFieldChangePossible', (datetime) => {
    nextFieldPossible.value = new Date(datetime)
    timerCount.value = (nextFieldPossible.value.getTime() - new Date().getTime()) / 1000
  })
})


function sendMessage() {
  if (message.value === undefined || message.value.length === 0) {
    return  
  }

  socket.emit("newMessage", message.value)
  message.value = ''
}

function setField(row: number, column: number) {
  if (nextFieldPossible.value !== null && (nextFieldPossible.value.getTime() - new Date().getTime()) > 0) {
    return
  }

  socket.emit('changeField', row, column, selectedColor.value)
}

function changeColor(color: string) {
  selectedColor.value = color
}

watch(timerCount, (value) => {
  if (value > 0) {
    setTimeout(() => {
      const number = (nextFieldPossible.value.getTime() - new Date().getTime()) / 1000
      timerCount.value = number <= 0 ? 0 : number
    }, 100);
  }
}, {immediate: true})
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
