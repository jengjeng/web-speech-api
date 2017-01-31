<template>
  <div id="app">
    <div class="chat" :class="{'recognition-support': recognition.support, 'synthesis-support': synthesis.support}">
      <div class="chat-title">
        <a href="https://konoesite.com" target="_blank">
          <h1>{{ user.name }}</h1>
          <h2>{{ user.description }}</h2>
        </a>
        <figure class="avatar">
          <img :src="user.image" /></figure>
      </div>
      <div class="messages">
        <div ref="messages" class="messages-content">
          <template v-for="message in messages">
            <message-them v-if="message.type === 'them'" :image="user.image" :content="message.content" :time="message.time"></message-them>
            <message-me v-if="message.type === 'me'" :content="message.content"></message-me>
          </template>
          <message-loading v-if="loading" :image="user.image"></message-loading>
        </div>
      </div>
      <form class="message-box" @submit.prevent="meTalk(input)">
        <div class="microphone" v-if="recognition.support" :class="{'is-listening': isListening }" @click="startListenVoiceCommands">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
            <path d="M12 16c2.206 0 4-1.795 4-4v-6c0-2.206-1.794-4-4-4s-4 1.794-4 4v6c0 2.205 1.794 4 4 4z"></path>
            <path d="M19 12v-2c0-0.552-0.447-1-1-1s-1 0.448-1 1v2c0 2.757-2.243 5-5 5s-5-2.243-5-5v-2c0-0.552-0.447-1-1-1s-1 0.448-1 1v2c0 3.52 2.613 6.432 6 6.92v1.080h-3c-0.553 0-1 0.447-1 1s0.447 1 1 1h8c0.553 0 1-0.447 1-1s-0.447-1-1-1h-3v-1.080c3.387-0.488 6-3.4 6-6.92z"></path>
          </svg>
        </div>
        <input ref="input" v-model="input" type="text" class="message-input" placeholder="Type message...">
        <button type="submit" class="message-submit">Send</button>
      </form>
    </div>
    <p id="P_1">
      Made with love by <a href="https://konoesite.com/" id="A_3">konoesite.com</a>
    </p>
  </div>
</template>
<script>
  import chatToken from './chatToken.private.js' // your chat api token

  import MessageMe from './MessageMe.vue'
  import MessageThem from './MessageThem.vue'
  import MessageLoading from './MessageLoading.vue'

  export default {
    name: 'app',
    data: () => ({
      user: {
        name: 'Wittawat Patcharinsak',
        description: '@KONOE',
        image: 'https://cdn-images-1.medium.com/fit/c/180/180/1*HuXi-m9Iov60QuZiFdJHyA.jpeg'
      },
      synthesis: {
        support: false,
        obj: null,
        config: {
          th: {
            lang: 'th-TH',
            pitch: 2,
            voiceIndex: 0
          },
          en: {
            lang: 'en-US',
            pitch: 2,
            voiceIndex: 6
          }
        }
      },
      recognition: {
        support: false,
        obj: null,
        config: {
          lang: 'th-TH',
          interimResults: false,
          maxAlternatives: true
        }
      },
      messages: [],
      input: '',
      loading: false,
      isListening: false,
      sessionId: Math.random().toString(36).slice(2),
      isChatLive: true,
      chatToken: chatToken
    }),
    created: function () {
      if ('SpeechSynthesisUtterance' in window) {
        this.synthesis.support = true
      }

      var SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition ||  undefined
      if (SpeechRecognition) {
        this.recognition.support = true
        this.recognition.SpeechRecognition = SpeechRecognition
      }
    },
    mounted: function () {
      var self = this
      this.$nextTick(function () {
        self.themTalk(self.synthesis.support ? 'สวัสดี' : 'Your browser does not support for Speech Synthesis', !
          self.synthesis.support,
          function () {
            self.$refs.input.focus()
          })
      })
    },
    watch: {
      loading: function () {
        this.onCompleteTalk()
      }
    },
    methods: {
      createMessage: function (type, message) {
        return {
          type: type,
          content: message,
          time: new Date()
        }
      },
      themTalk: function (message, force, callback) {
        if (this.loading || !message.trim()) return
        var self = this
        self.loading = true
        var putMessage = function (message) {
          var msg = self.createMessage('them', message)
          self.messages.push(msg)
          self.speak(message)
          self.loading = false
          self.onCompleteTalk()
        }
        if (!force && self.isChatLive && !self.checkThaiLanguage(message)) {
          self.getChatResponse(message).then(function (data) {
            putMessage(data || message)
            callback && callback()
          }).catch(function () {
            self.isChatLive = false
            putMessage(message)
            callback && callback()
          })
        } else {
          setTimeout(function () {
            putMessage(message)
            callback && callback()
          }, Math.random() * 2000 + 500)
        }
      },
      meTalk: function (message) {
        if (!message.trim()) return
        var self = this
        var msg = this.createMessage('me', message)
        self.messages.push(msg)
        self.resetForm()
        self.onCompleteTalk()
        setTimeout(function () {
          self.themTalk(message)
        }, Math.random() * 500 + 500)
      },
      onCompleteTalk: function () {
        var self = this
        self.$nextTick(function () {
          self.$refs.messages.scrollTop = this.$refs.messages.scrollHeight
        })
      },
      resetForm: function () {
        this.input = ''
      },
      speak: function (message) {
        if (!this.synthesis.support) return
        var self = this
        var langKey = this.checkThaiLanguage(message) ? 'th' : 'en'
        var config = this.synthesis.config[langKey]

        self.$refs.input.onchange = function () {
          self.synthesis.obj = new SpeechSynthesisUtterance()
          for (var k in config) {
            if (k !== 'voiceIndex')
              self.synthesis.obj[k] = config[k]
          }
          self.synthesis.obj.voice = speechSynthesis.getVoices().filter(function (voice) {
            return voice.lang.indexOf(langKey) >= 0;
          })[config.voiceIndex];
          self.synthesis.obj.text = message
          speechSynthesis.speak(self.synthesis.obj)
        }
        self.$refs.input.onchange()
        self.$refs.input.onchange = undefined
      },
      startListenVoiceCommands: function () {
        if (this.isListening) return
        var self = this
        self.isListening = true
        this.recognition.obj = new this.recognition.SpeechRecognition()
        for (var k in this.recognition.config) {
          this.recognition.obj[k] = this.recognition.config[k]
        }
        self.recognition.obj.start()
        self.recognition.obj.onresult = function (event) {
          var result = event.results[0][0].transcript
          self.meTalk(result)
        }
        self.recognition.obj.onend = function () {
          self.isListening = false
          self.recognition.obj.stop()
        }
      },
      checkThaiLanguage(message) {
        return /[\u0E01-\u0E5B]/.test(message)
      },
      getChatResponse: function (text) {
        return axios.post('https://api.api.ai/v1/query', {
          query: text,
          lang: "en",
          sessionId: this.sessionId
        }, {
          headers: {
            'Accept': 'application/json',
            "Authorization": "Bearer " + this.chatToken
          }
        }).then(function (response) {
          return response.data.result.speech || response.data.result.parameters.simplified
        })
      }
    },
    components: {
      MessageMe,
      MessageThem,
      MessageLoading
    }
  }
</script>
<style>
  @import './assets/style.css';
</style>