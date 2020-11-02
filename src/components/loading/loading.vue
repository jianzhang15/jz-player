<template>
  <transition name="fade">
    <section>
      <div class="loading">
        <!-- <img width="24px" height="24px" src="../images/timg.gif">
                <p class="desc">{{title}}</p> -->
        <span></span>
        <span></span>
        <span></span>
        <span></span>
        <span></span>
      </div>
    </section>
  </transition>
</template>
<script>
export default {
  props: {
    title: {
      type: String,
      default: "正在载入..."
    }
  }
};
</script>
<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}
section {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background:#fff;
  z-index: 999;
  display: flex;
  align-items: center;
  justify-content: center;
}
.loading {
  width: 100%;
  text-align: center;
  /* flex-direction: column; */
}
.loading span {
  display: inline-block;
  width: 8px;
  height: 100%;
  border-radius: 4px;
  background: lightgreen;
  animation: load 1s ease infinite;
  margin-left: 5px;
}
@keyframes load {
  0%,
  100% {
    height: 10px;
    background: lightgreen;
  }
  50% {
    height: 30px;
    margin: -5px 0;
    background: lightblue;
  }
}
.loading span:nth-child(2) {
  animation-delay: 0.2s;
}
.loading span:nth-child(3) {
  animation-delay: 0.4s;
}
.loading span:nth-child(4) {
  animation-delay: 0.6s;
}
.loading span:nth-child(5) {
  animation-delay: 0.8s;
}
</style>
