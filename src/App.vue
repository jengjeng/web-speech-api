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
      <form ref="speakEvent" class="message-box" @submit.prevent="meTalk(input)">
        <div class="microphone" v-if="recognition.support" :class="{'is-listening': isListening }" @click="startListenVoiceCommands">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
            <path d="M12 16c2.206 0 4-1.795 4-4v-6c0-2.206-1.794-4-4-4s-4 1.794-4 4v6c0 2.205 1.794 4 4 4z"></path>
            <path d="M19 12v-2c0-0.552-0.447-1-1-1s-1 0.448-1 1v2c0 2.757-2.243 5-5 5s-5-2.243-5-5v-2c0-0.552-0.447-1-1-1s-1 0.448-1 1v2c0 3.52 2.613 6.432 6 6.92v1.080h-3c-0.553 0-1 0.447-1 1s0.447 1 1 1h8c0.553 0 1-0.447 1-1s-0.447-1-1-1h-3v-1.080c3.387-0.488 6-3.4 6-6.92z"></path>
          </svg>
        </div>
        <input ref="input" v-model="input" type="text" class="message-input" placeholder="Type message...">
        <button ref="submit" type="submit" class="message-submit">Send</button>
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
            pitch: 2
          },
          en: {
            lang: 'en-US',
            pitch: 2
          }
        },
        lastMsg: {}
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
      chatToken: chatToken,
      isMobile: checkMobile()
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
          time: new Date(),
          id: Math.random().toString(36).slice(2)
        }
      },
      themTalk: function (message, force, callback) {
        if (this.loading || !message.trim()) return
        var self = this
        self.loading = true
        var putMessage = function (message) {
          var msg = self.createMessage('them', message)
          self.messages.push(msg)
          self.speak(msg)
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
      speak: function (msg) {
        if (!this.synthesis.support) return
        var self = this
        var langKey = this.checkThaiLanguage(msg.content) ? 'th' : 'en'
        var config = this.synthesis.config[langKey]

        self.synthesis.message = msg.content

        self.$refs.speakEvent.onclick = function (e) {
          if (!self.isMobile) {
            self.synthesis._init = true
          }
          if (e !== 'force') {
            if (self.synthesis._init) {
              return
            } else {
              self.synthesis._init = true
            }
          }
          // if (self.synthesis.lastMsg.id === msg.id) {
          //   self.synthesis.message = ''
          // }
          self.synthesis.obj = new SpeechSynthesisUtterance()
          for (var k in config) {
            self.synthesis.obj[k] = config[k]
          }
          self.synthesis.obj.voice = langKey === 'th' ? self.getThaiVoice() : self.getEngVoice()
          self.synthesis.obj.text = self.synthesis.message
          setTimeout(function() {
            speechSynthesis.speak(self.synthesis.obj)
            self.synthesis.lastMsg = msg
          }, 200)
        }
        self.$refs.speakEvent.onclick('force')
      },
      getThaiVoice: function () {
        return this.filterVoice(v => v.lang === 'th-TH')
          || this.filterVoice(v => v.lang.indexOf('th') >= 0)
      },
      getEngVoice: function () {
        return this.filterVoice(v => v.name === 'Samantha')
      },
      filterVoice: function (cb) {
        return speechSynthesis.getVoices().filter(cb)[0]
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

  function checkMobile() {
    return /(android|bb\d+|meego).+mobile|avantgo|bada\/|blackberry|blazer|compal|elaine|fennec|hiptop|iemobile|ip(hone|od)|ipad|iris|kindle|Android|Silk|lge |maemo|midp|mmp|netfront|opera m(ob|in)i|palm( os)?|phone|p(ixi|re)\/|plucker|pocket|psp|series(4|6)0|symbian|treo|up\.(browser|link)|vodafone|wap|windows (ce|phone)|xda|xiino/i.test(navigator.userAgent) 
    || /1207|6310|6590|3gso|4thp|50[1-6]i|770s|802s|a wa|abac|ac(er|oo|s\-)|ai(ko|rn)|al(av|ca|co)|amoi|an(ex|ny|yw)|aptu|ar(ch|go)|as(te|us)|attw|au(di|\-m|r |s )|avan|be(ck|ll|nq)|bi(lb|rd)|bl(ac|az)|br(e|v)w|bumb|bw\-(n|u)|c55\/|capi|ccwa|cdm\-|cell|chtm|cldc|cmd\-|co(mp|nd)|craw|da(it|ll|ng)|dbte|dc\-s|devi|dica|dmob|do(c|p)o|ds(12|\-d)|el(49|ai)|em(l2|ul)|er(ic|k0)|esl8|ez([4-7]0|os|wa|ze)|fetc|fly(\-|_)|g1 u|g560|gene|gf\-5|g\-mo|go(\.w|od)|gr(ad|un)|haie|hcit|hd\-(m|p|t)|hei\-|hi(pt|ta)|hp( i|ip)|hs\-c|ht(c(\-| |_|a|g|p|s|t)|tp)|hu(aw|tc)|i\-(20|go|ma)|i230|iac( |\-|\/)|ibro|idea|ig01|ikom|im1k|inno|ipaq|iris|ja(t|v)a|jbro|jemu|jigs|kddi|keji|kgt( |\/)|klon|kpt |kwc\-|kyo(c|k)|le(no|xi)|lg( g|\/(k|l|u)|50|54|\-[a-w])|libw|lynx|m1\-w|m3ga|m50\/|ma(te|ui|xo)|mc(01|21|ca)|m\-cr|me(rc|ri)|mi(o8|oa|ts)|mmef|mo(01|02|bi|de|do|t(\-| |o|v)|zz)|mt(50|p1|v )|mwbp|mywa|n10[0-2]|n20[2-3]|n30(0|2)|n50(0|2|5)|n7(0(0|1)|10)|ne((c|m)\-|on|tf|wf|wg|wt)|nok(6|i)|nzph|o2im|op(ti|wv)|oran|owg1|p800|pan(a|d|t)|pdxg|pg(13|\-([1-8]|c))|phil|pire|pl(ay|uc)|pn\-2|po(ck|rt|se)|prox|psio|pt\-g|qa\-a|qc(07|12|21|32|60|\-[2-7]|i\-)|qtek|r380|r600|raks|rim9|ro(ve|zo)|s55\/|sa(ge|ma|mm|ms|ny|va)|sc(01|h\-|oo|p\-)|sdk\/|se(c(\-|0|1)|47|mc|nd|ri)|sgh\-|shar|sie(\-|m)|sk\-0|sl(45|id)|sm(al|ar|b3|it|t5)|so(ft|ny)|sp(01|h\-|v\-|v )|sy(01|mb)|t2(18|50)|t6(00|10|18)|ta(gt|lk)|tcl\-|tdg\-|tel(i|m)|tim\-|t\-mo|to(pl|sh)|ts(70|m\-|m3|m5)|tx\-9|up(\.b|g1|si)|utst|v400|v750|veri|vi(rg|te)|vk(40|5[0-3]|\-v)|vm40|voda|vulc|vx(52|53|60|61|70|80|81|83|85|98)|w3c(\-| )|webc|whit|wi(g |nc|nw)|wmlb|wonu|x700|yas\-|your|zeto|zte\-/i.test(navigator.userAgent.substr(0,4))
  }
</script>
<style>
  @import './assets/style.css';
</style>