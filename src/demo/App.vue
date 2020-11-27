<template>
  <div id="app">
    <div class="container">
      <h1 align="center">基于Vue-APlayer修改</h1>
      <h3 @click="init">点击播放</h3>
      <h3 @click="stop">点击停止</h3>
      <h3 @click="playMusic1">切换播放列表(1条音频)</h3>
      <h3 @click="playMusic2">切换播放列表(2条音频)</h3>
      <aplayer
        v-if="showPlayer"
        ref="player"
        autoplay
        :music="list3[0]"
        :list="list3"
        repeat="REPEAT.REPEAT_ONCE"
        listFolded
        float
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
            src: 'https://m10.music.126.net/20201127200739/02e17c0534167fc65484b08039e7d21c/yyaac/obj/wonDkMOGw6XDiTHCmMOi/2183399114/e1a7/6385/f7cf/4f70fdc1a0fb9ab704268c0fb7bbfe6b.m4a',
          },
          {
            title: '前前前世',
            src: '/api/file/download?audioId=5fbe655bbe8d985ef85af7b7',
          }
        ],
        showPlayer:false
      }
    },
    methods:{
      init (){
        this.showPlayer=true
      },
      stop (){
        this.$refs['player'].stop()
      },
      playMusic1 (){
        this.list3=[
          {
            title: '前前前世',
            src: '/api/file/download?audioId=5fbe655bbe8d985ef85af7b7',
          }
        ]
      },
      playMusic2 (){
        this.list3=[
          {
            title: '前前前世',
            src: '/api/file/download?audioId=5fbe655bbe8d985ef85af7b7',
          },
          {
            title: '前前前世',
            src: '/api/file/download?audioId=5fbe655bbe8d985ef85af7b5',
          }
        ]
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