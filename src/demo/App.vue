<template>
  <div id="app">
    <div class="container">
      <h1 align="center">基于Vue-APlayer修改</h1>
      <h3 @click="playMusic">点击播放</h3>
      <aplayer
        ref="player"
        :music="list3[0]"
        :list="list3"
        listFolded
      />
    </div>
  </div>
</template>
<script>
  import Aplayer from '../vue-aplayer.vue'

  export default {
    name: 'App',
    components: {
      Aplayer,
    },
    data () {
      return {
        version: VERSION,
        volume: 1,
        muted: false,
        music3: null,
        list3: [
          {
            title: '前前前世',
            src: 'https://m10.music.126.net/20201102203723/ddcb87f2a06d86e7cb28501d262573ea/yyaac/0f0c/0e0b/065b/8f6e502bbb9acfdddce21126d33d5bf2.m4a',
          },
          {
            title: '光るなら.m3u8',
            src: 'https://m10.music.126.net/20201102203723/ddcb87f2a06d86e7cb28501d262573ea/yyaac/0f0c/0e0b/065b/8f6e502bbb9acfdddce21126d33d5bf2.m4a',
          },
          {
            title: '回レ！雪月花',
            src: 'https://m10.music.126.net/20201102203723/ddcb87f2a06d86e7cb28501d262573ea/yyaac/0f0c/0e0b/065b/8f6e502bbb9acfdddce21126d33d5bf2.m4a',
          },
          {
            title: 'あっちゅ～ま青春!',
            src: 'https://m10.music.126.net/20201102203723/ddcb87f2a06d86e7cb28501d262573ea/yyaac/0f0c/0e0b/065b/8f6e502bbb9acfdddce21126d33d5bf2.m4a',
          },
        ],
      }
    },
    methods:{
      playMusic (){
        this.$refs['player'].toggle()
      }
    }
  }
</script>
<style>
  html, body {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
    overflow: hidden;
  }

  body {
    font-family: Source Sans Pro, 'PingFang SC', 'Helvetica Neue', Helvetica, Arial, sans-serif;
    background: linear-gradient(135deg, rgb(65, 184, 131), rgb(74, 156, 238));
    color: #fff;
    overflow-y: auto;
  }

  #app {
    text-align: center;
  }

  .container {
    max-width: 32rem;
    margin-left: auto;
    margin-right: auto;
    margin-bottom: 50px;
    padding: 0 15px;
  }

  h1 {
    font-size: 48px;
    margin: 30px 0 10px;
  }

  h3 {
    font-size: 24px;
  }

  input {
    background-color: transparent;
    outline: none;
    border: none;
    color: inherit;
    font-size: inherit;
    text-align: center;
    border-bottom: 1px solid rgba(255, 255, 255, .5);
  }

  a {
    color: #35495e;
    text-decoration: none;
  }

  p {
    font-size: 18px;
  }
</style>