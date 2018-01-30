<template>
  <div>
    <h1>kingkong tapping</h1>
    <input v-model="targetURL">輸入網址</input>
    <button @click="run">載入</button>
    <input type="checkbox" v-model="showDev">開發者資訊</input>
    <p>
      {{authenticated ? 'socket 連接成功' : 'socket 尚未連接'}}
    </p>
    <div v-if="showDev">
      <p>
        {{payload}}
      </p>
      <p>
        liveKey {{payload.liveKey}}
      </p>
      <p>
        liveId {{payload.liveId}}
      </p>
      <p>
        {{token}}
      </p>
    </div>
    <table>
      <tr>
        <th>名稱</th>
        <th>訊息</th>
      </tr>
      <tr v-for="message in messages">
        <th>{{message.name}}</th>
        <th>{{message.message}}</th>
      </tr>
    </table>
    <!-- <button v&#45;if="token" @click="connectSocket">connect socket</button> -->
  </div>
</template>

<script>

import * as uuidv4 from 'uuid/v4'
import * as jwt from 'jsonwebtoken'

export default {
  data () {
    return {
      targetURL: null,
      payload: {},
      ws: null,
      showDev: false,
      token: null,
      messages: [],
      authenticated: false
    }
  },
  computed: {
    uuid: function () {
      return uuidv4().replace(/\-/g, '')
    },
    name: function () {
      if (!this.uuid) {
        return
      }
      return `訪客${this.uuid.slice(-3)}`
    }
  },
  methods: {
    async connectSocket (address) {
      const ws = new WebSocket('wss://cht.ws.kingkong.com.tw/chat_nsp/?EIO=3&transport=websocket')
      this.ws = ws
      ws.addEventListener('open', function open () {
        console.log('[socket] connected')
        ws.send('40/chat_nsp,')
      })
      ws.addEventListener('close', function close () {
        console.log('[socket] disconnected :(')
      })
      ws.addEventListener('message', (event) => {
        console.log('[socket][received]', typeof event.data, event.data)
        if (event.data.startsWith('42/chat_nsp')) {
          let data = JSON.parse(event.data.slice(12))
          if (data[0] === 'authenticated') {
            this.authenticated = data[1]
          }
          if (data[0] !== 'msg') {
            return
          }
          console.log('got message data')
          this.messages.push({
            name: data[1].name,
            message: data[1].msg
          })
        }
        if (event.data.startsWith('0')) {
          let payload = `42/chat_nsp,["authentication",{"live_id":"${this.payload.data.live_id}","anchor_pfid":${this.payload.anchorId},"token":"${this.token}","client_type":"web"}]`
          console.log('going with payload', payload)
          ws.send(payload)
        }
      })
    },
    async generateToken () {
      return jwt.sign({
        live_id: this.payload.data.live_id,
        pfid: this.uuid,
        name: this.name,
        lv: 1,
        from: 1,
        rom_seq: 1,
        channel_id: 1,
        client_type: 'web'
      }, this.payload.data.live_key, {
        algorithm: 'HS256'
      })
    },
    async run () {
      if (!this.targetURL) {
        return
      }
      let blob = await this.fetchRoomInfo(this.targetURL)
      this.payload = await this.parseBlob(blob)
      this.token = await this.generateToken()
      await this.connectSocket()
    },
    async fetchRoomInfo (targetURL) {
      let result = await fetch(`https://cors-anywhere.herokuapp.com/${targetURL}`)
      return await result.blob()
    },
    async readBlob (blob) {
      return new Promise((resolve, reject) => {
        var fr = new FileReader()
        fr.onload = () => {
          resolve(fr.result)
        };
        fr.readAsText(blob)
      })
    },
    async parseBlob (blob) {
      let htmlString = await this.readBlob(blob)
      let parser = new DOMParser()
      let html = parser.parseFromString(htmlString, 'text/html')
      let viewLive = html.querySelector('.view-live')
      let roomInfo = viewLive.getAttribute('data-roomInfo')
      return JSON.parse(roomInfo)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
