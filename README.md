## 简介
本项目是基于vue-aplayer二次开发的适用于PC端的采用wavesurfer.js的音乐播放器，可以参考这里[vue-aplayer](https://github.com/SevenOutman/vue-aplayer)，原版的Vue-aplayer采用audio标签已被抛弃
## 安装

### Node

```
$ yarn add jz-player
```

或者如果你选择 `npm`

```
$ npm i jz-player
```
## 示例图

![Screenshot](https://github.com/jianzhang15/jz-player/blob/main/demo.png)

## 使用

```HTML
<aplayer autoplay
  :music="{
    title: 'secret base~君がくれたもの~',
    artist: 'Silent Siren',
    src: 'http://music.163.com/song/media/outer/url?id=447925558.mp3',
    pic: 'https://moeplayer.b0.upaiyun.com/aplayer/secretbase.jpg'
  }"
/>
```

```JS
// ES6
import Aplayer from 'vue-aplayer'

new Vue({
    components: {
        Aplayer
    }
})
```



### Props

| 名称 | 类型 | 默认值 | 说明 |
| ---- | ---- | ------- | ----------- |
| music| Object | *必需* | 当前播放的音乐。 具体请看[音乐信息](https://github.com/SevenOutman/vue-aplayer/blob/develop/docs/README.zh-CN.md#音乐信息) |
| list | Array | `[]` | 播放列表。如果 list 不是空数组，播放列表就会显示出来，即使 list 中只有一首歌并且它和 music 一样 |
| mini | Boolean | `false` | 迷你模式 |
| float | Boolean | `false` | 浮动模式。你可以在页面上随意拖放你的播放器 |
| showLrc | Boolean | `false` | 是否显示歌词 |
| mutex | Boolean | `true` | 是否在该播放器播放时暂停其他播放器 |
| theme | String | `'#41b883'` | 主题色。如果当前歌曲也设置了 `theme` 则以歌曲的为准 |
| shuffle | Boolean | `false` | 随机播放 |
| repeat | String | `'REPEAT.REPEAT_ALL'` | 循环模式。值可以是 `'REPEAT.REPEAT_ONE'`（单曲循环）`'REPEAT.REPEAT_ONCE'`（列表播放1遍）`'REPEAT.REPEAT_ALL'`（列表循环）或者 `'REPEAT.NO_REPEAT'`（不循环） |
| listMaxHeight | String | *none* | 播放列表面板最大高度 |
| listFolded | Boolean | `false` | 默认收起播放列表 |
| autoplay | Boolean | `false`  | 自动播放。如果多个 mutex 播放器设置了 autoplay，只有第一个会自动播放（因）|
| muted | Boolean | `false`  | 静音 |
| volume | Number | `0.8`  | 播放音量，0为最小，1为最大 |

> 如果你是用的是 Vue@2.3.0+, 你可以在 `music` `shuffle` 和 `repeat` 上使用 [`.sync` 修饰符](https://cn.vuejs.org/v2/guide/components.html#sync-%E4%BF%AE%E9%A5%B0%E7%AC%A6)，`muted` 和 `volume` 也可以使用 `.sync` 修饰符，你可以利用这一点做一些自定义的控制


### Props.music

`music` props 包含了当前播放歌曲的如下信息。

| 属性 | 默认值 | 说明 |
| -------- | ------- | ----------- |
| src | *必需* | 音频文件的 URL |
| title | `'Untitled'` | 歌曲名称 |
| artist | `'Unknown'` | 演唱者 |
| pic | *none* | 封面图片 URL |
| lrc | *none* | LRC 歌词或者歌词文件的 URL |
| theme | *none* | 歌曲的主题色，会覆盖播放器的主题色 |



### Events

| 事件 | 参数 | 说明 |
| -------- | ------- | ----------- |
| play | `currentMusic` | 播放回调事件，会返回当前播放的音频Object |
| pause | `currentMusic` | 暂停回调事件，会返回当前播放的音频Object |
| stop | `currentMusic` | 停止回调事件，会返回当前播放的音频Object |
| end | `currentMusic` | 播放完回调事件，仅当repeat模式为“REPEAT.NO_REPEAT”生效，会返回当前播放的音频Object |
## 进阶使用

### 自适应主题色

从 `v1.3.0` 开始, 如果你将一首歌的 `theme` 值设为 `'pic'`, Vue-APlayer 会从它的封面图片中提取颜色作为主题色.
你也可以直接将 Vue-APlayer 的 `theme` prop 设为 `'pic'`, 这样所有的歌曲都会使用自适应主题色

你只需将 [color-thief](https://github.com/lokesh/color-thief) 库加入页面中.
> 注意 color-thief **无法正常使用 npm 安装**

```
<!-- 或者你选择的其他 CDN -->
<script src="https://cdn.jsdelivr.net/npm/colorthief@2.0.2/src/color-thief.js"></script>
```


### `disableVersionBadge`

Vue-APlayer 默认会在控制台打印出当前的版本标识，如果你想要禁用它，可以将 `disableVersionBadge` 设为 `true`。

```js
import VueAPlayer from 'vue-aplayer'
VueAPlayer.disableVersionBadge = true
```


### Slots

- `slot="display"`

这个 slot 代表播放器主体上显示的内容, 默认是滚动歌词.

这个 slot 中的组件会接收两个 props: `currentMusic` 和 `playStat`.
