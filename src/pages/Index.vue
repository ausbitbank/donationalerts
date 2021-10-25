<template>
  <q-page class="flex flex-center">
    <div v-if="hideUI === 'true'" class="text-center text-white text-h6" style="">
      <div v-if="showAlert === true">
      <transition appear :enter-active-class="anim1" :leave-active-class="anim2">
        <div class="text-effect q-ma-sm q-pa-sm"><div class="neon" :data-text="alert.title">{{ alert.title }}</div><div class="gradient"></div><div class="spotlight"></div></div>
      </transition>
      <transition appear :enter-active-class="anim1" :leave-active-class="anim2">
        <img :src="image" />
      </transition>
      <transition appear :enter-active-class="anim1" :leave-active-class="anim2">
      <div class="text-title bg-black q-ma-sm q-pa-sm" style="font-size: 50px">
        {{ alert.user }} sent {{ alert.amount.split(' ')[0] }} <q-avatar size="md" class="q-ma-xs"><img src="https://hive.ausbit.dev/statics/hive.svg" /></q-avatar>
      </div>
      </transition>
      <transition appear :enter-active-class="anim1" :leave-active-class="anim2">
      <div style="font-size: 40px" class="memo q-ma-md q-pa-sm bg-black" v-if="alert.memo">
        <i>"{{ alert.memo }}"</i>
      </div>
      </transition>
      </div>
    </div>
    <div v-else-if="hideUI !== 'true'">
      <q-input label="Hive username" v-model="username" />
      <q-input label="Alert image" v-model="image" />
      <q-img :src="image" />
      <q-input label="Alert sound" v-model="sound" />
      <q-btn flat @click="playSound()" icon="play_arrow" title="Test alert sound" />
      <q-input label="Title" v-model="title" />
      <q-input label="Min amount to trigger alert" v-model="minAlert" />
      <q-input label="Min amount to display memo" v-model="minMemo" />
      <q-input label="Min amount to trigger text to speech" v-model="minTTS" />
      <q-input label="Ignore transfers from these accounts" v-model="ignoreAccounts" />
      <q-input label="Entry animation" v-model="animation1" />
      <q-input label="Exit animation" v-model="animation2" />
      <div>^^ Get animation styles from <a href="https://animate.style/" target="_blank">animate.style</a></div>
      <q-btn label="Get custom link for OBS browser source" icon="link" color="primary">
        <q-popup-proxy class="text-center">
          <q-card flat bordered class="q-ma-sm q-pa-sm">
            This link has all of your custom settings embedded.<br />
            Add this to your OBS scene as a browser source :
            <q-input v-model="url" readonly />
            <q-btn icon="content_copy" color="primary" label="Copy URL to clipboard" @click="copyClipboard(url)" /><br />
            And add this to the custom css to make the background transparent<br />
            <q-input v-model="css" readonly />
            <q-btn icon="content_copy" color="primary" label="Copy CSS to clipboard" @click="copyClipboard(css)" /><br />
          </q-card>
        </q-popup-proxy>
      </q-btn><br />
      <q-btn label="Test text to speech" @click="textToSpeech('Testing text to speech, ermagerd')" color="orange" icon="tty" v-if="false" />
      <q-btn label="test dono TTS" @click="newDonation(alert.user,alert.amount,alert.memo)" color="orange" icon="tty" />
    </div>
  </q-page>
</template>
<script>
const { Client } = require("@hiveio/dhive")
import { onMounted, defineComponent, ref } from 'vue'
import { useQuasar, copyToClipboard } from 'quasar'
import { useRouter, useRoute } from 'vue-router'
export default defineComponent({
  name: 'PageIndex',
  async mounted () {
    if (this.hideUI === true) { console.log('ui hidden') }
    while (true && this.timerStarted && this.username !== '') {
      await this.transferCheck()
      console.log('Check count: ' + this.checkCount)
      this.checkCount += 1
    }
  },
  setup () {
    const client = new Client(["https://rpc.ausbit.dev", "https://api.hive.blog", "https://api.openhive.network"])
    const $q = useQuasar()
    const router = useRouter()
    const route = useRoute()
    return {
      username: route.query.username || ref(''),
      image: route.query.image || ref('https://c.tenor.com/ckMRF2IunhwAAAAC/money-shutup.gif'),
      sound: route.query.sound || ref('https://files.ausbit.dev/mixkit-magical-coin-win-1936.wav'),
      minAlert: route.query.minAlert || ref(0.01),
      minMemo: route.query.minMemo || ref(0.1),
      minTTS: route.query.minTTS || ref(1),
      css: 'body { background-color: rgba(0, 0, 0, 0); margin: 0px auto; overflow: hidden; }',
      ignoreAccounts: route.query.ignoreAccounts || ref('ocdb,reward.app'),
      title: route.query.title || ref('Donation!'),
      hideUI: route.query.hideUI || ref(false),
      animation1: route.query.animation1 || ref('fadeIn'),
      animation2: route.query.animation2 || ref('fadeOut'),
      showAlert: ref(false),
      timerStarted: ref(true),
      checkCount: ref(1),
      alertDelay: ref(15000), // 15 seconds before alert dissapears
      checkDelay: ref(60000), // 60 seconds between account checks
      alreadyPlayedAlerts: ref([]), // id's of already played alerts
      alert: {
        title: 'Donation!',
        user: 'ausbitbank',
        memo: 'Great stream keep it up!',
        amount: '10'
      },
      transferHistory: {
        current: [],
        latest: null
      },
      playSound () {
        let sound = new Audio(this.sound)
        sound.load()
        sound.play()
      },
      async getTransfers () {
        this.timerStarted = true
        var h = await client.call('condenser_api', 'get_account_history', [this.username, -1, 5, "4", null]) // Get transfer operations only
        if (this.transferHistory.latest !== null) { // Not the first loop
          if (h[h.length - 1][0] > this.transferHistory.latest[0]) { // If the last saved txid is different to the most recent
            console.log('new transfers')
            h.forEach( i => {
              var id = i[0]
              var op = i[1].op[1]
              var amount = parseInt(op.amount.split(' ')[0])
              if (!this.alreadyPlayedAlerts.includes(id)) { // We haven't seen this alert already
                if (id > this.transferHistory.latest[0]) { // If ID is greater then the last known transfer
                  if (op.to === this.username) {// And is an incoming transfer
                    if (amount >= this.minAlert) { // Amount is above minimum for alert
                      console.log(op)
                      this.playSound()
                      this.alert.user = op.from
                      this.alert.amount = op.amount
                      this.alert.memo = op.memo
                      this.showAlert = true
                      if (amount >= this.minMemo && op.memo !== '') { // Amount is above minimum for memo
                        this.alert.memo = op.memo
                        if (amount >= this.minTTS) { // Amount is above minimum for tts
                          var tts = this.alert.user + ' sent ' + amount + ' hive ... ' + this.alert.memo
                          console.log('Play tts: "' + tts + '"')
                          this.textToSpeech(tts)
                        }
                      }
                      this.alreadyPlayedAlerts.push(id)
                      this.hideAlert()
                    }
                  }
                }
              }
            })
            this.transferHistory.current = h
            this.transferHistory.latest = this.transferHistory.current[this.transferHistory.current.length - 1]
          }
        } else { // Is the first loop, save data but no alerts yet
          console.log('first account check for ' + this.username)
          this.transferHistory.current = h
          this.transferHistory.latest = this.transferHistory.current[this.transferHistory.current.length - 1]
          h.forEach( i => { this.alreadyPlayedAlerts.push(i[0]) })
        }
      },
      delay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms))
      },
      async hideAlert() {
        await this.delay(this.alertDelay)
        this.showAlert = false
      },
      async transferCheck() {
        await this.getTransfers()
        await this.delay(this.checkDelay)
      },
      copyClipboard (what) {
        copyToClipboard(what)
          .then(() => { $q.notify("Copied to clipboard") })
      },
      newDonation(username, amount, memo) {
        this.showAlert = true
        this.textToSpeech(username + ' sent ' + amount + ' hive')
        this.textToSpeech(memo)
      },
      textToSpeech(what) {
        window.speechSynthesis.speak(new SpeechSynthesisUtterance(what))
      }
    }
  },
  computed: {
    url() { return 'https://donationalert.ausbit.dev/?username=' + this.username + '&image=' + this.image + '&sound=' + this.sound + '&minAlert=' + this.minAlert + '&minMemo=' + this.minMemo + '&minTTS=' + this.minTTS + '&ignoreAccounts=' + this.ignoreAccounts + '&anim1=' + this.animation1 + '&anim2=' + this.animation2 + '&hideUI=true' },
    anim1() { return 'animated ' + this.animation1 },
    anim2() { return 'animated ' + this.animation2 }
  }
})
</script>
<style>
@import url('https://fonts.googleapis.com/css2?family=Roboto+Slab:wght@300&display=swap');
@import url('https://fonts.googleapis.com/css2?family=Dancing+Script&family=Roboto+Slab:wght@300&display=swap');
.memo {
  font-family: 'Dancing Script', cursive;
  font-size: 50px
}
.text-effect {
  overflow: hidden;
  position: relative;
  filter: contrast(110%) brightness(190%);
}

.neon {
  position: relative;
  background: black;
  color: transparent;
}
.neon::before, .neon::after {
  content: attr(data-text);
  color: white;
  filter: blur(0.02em);
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}
.neon::after {
  mix-blend-mode: difference;
}

.gradient,
.spotlight {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  pointer-events: none;
  z-index: 10;
}

.gradient {
  background: linear-gradient(45deg, red, blue);
  mix-blend-mode: multiply;
}

.spotlight {
  -webkit-animation: light 5s infinite linear;
          animation: light 5s infinite linear;
  background: radial-gradient(circle, white, transparent 25%) 0 0/25% 25%, radial-gradient(circle, white, black 25%) 50% 50%/12.5% 12.5%;
  top: -100%;
  left: -100%;
  mix-blend-mode: color-dodge;
}

@-webkit-keyframes light {
  100% {
    transform: translate3d(50%, 50%, 0);
  }
}

@keyframes light {
  100% {
    transform: translate3d(50%, 50%, 0);
  }
}
.neon {
  font: 700 80px "Lato", sans-serif;
  text-transform: uppercase;
  text-align: center;
  margin: 0;
}
.neon:focus {
  outline: none;
  border: 1px dotted white;
}

body {
  display: flex;
  min-height: 100vh;
  justify-content: center;
  align-content: center;
  align-items: center;
  font-family: 'Roboto Slab', serif;
}
</style>