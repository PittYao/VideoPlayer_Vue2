<template>
  <div>
    <video-player
      v-for="(item, index) in rtspList"
      :key="index"
      :rtspUrl="item.rtspUrl"
      :disableAudio="item.disableAudio"
      :play="item.play"
      :muted="item.muted"
      :full.sync="item.full"
      :class="{ selected: item.selected }"
      @click.native="selectedVideoElem(index)"
    />
    <div>
      <button @click="getRtspList">获取播放地址集合</button> |
      <button @click="switchVideo(true)">开始播放</button> |
      <button @click="switchVideo(false)">关闭播放</button> |
      <button @click="switchMuted(true)">静音</button> |
      <button @click="switchMuted(false)">取消静音</button> |
      <button @click="launchFull">全屏</button> |
    </div>
  </div>
</template>

<script>
import VideoPlayer from "@/components/VideoPlayer.vue";

export default {
  name: "App",
  components: {
    VideoPlayer,
  },
  data() {
    return {
      // 流播放集合
      rtspList: [],
      // 选中流集合
      selectedList: [],
    };
  },
  methods: {
    getRtspList() {
      this.rtspList = [
        {
          // rtspUrl: "rtsp://admin:hk123456@10.0.5.2:554",
          rtspUrl: "rtsp://admin:cebon61332433@192.168.99.215",
          disableAudio: false,
          play: false,
          selected: false,
          muted: false,
          full: false,
        },
        {
          // rtspUrl: "rtsp://admin:hk123456@10.0.6.2:554",
          rtspUrl: "rtsp://admin:61332433@192.168.99.117",
          disableAudio: true,
          play: true,
          selected: false,
          muted: false,
          full: false,
        },
      ];
    },
    // 添加节点到选中集合
    selectedVideoElem(index) {
      let videoElem = this.rtspList[index];
      if (videoElem !== undefined) {
        this.selectedList.includes(index) === true
          ? this.selectedList.shift(index)
          : this.selectedList.push(index);
        videoElem.selected = !videoElem.selected;
      }
    },
    // 切换 开始/关闭 播放
    switchVideo(play) {
      this.selectedList.forEach((index) => {
        this.rtspList[index].play = play;
      });

      this.clearSelected();
    },
    // 切换 静音 控制
    switchMuted(muted) {
      this.selectedList.forEach((index) => {
        this.rtspList[index].muted = muted;
      });

      this.clearSelected();
    },
    // 全屏
    launchFull() {
      const data = this.selectedList;

      if (data.length > 1 || data.length === 0) {
        alert("只能选择一个画面全屏");
        return;
      }

      const index = data[0];
      this.rtspList[index].full = true;

      this.clearSelected();
    },
    // 清除选中
    clearSelected() {
      this.clearSelectedStyle();
      this.selectedList = [];
    },
    // 清除选中样式
    clearSelectedStyle() {
      this.rtspList.forEach((item) => {
        item.selected = false;
      });
    },
  },
};
</script>


<style lang="scss" scoped>
.selected {
  border: 1px solid red;
}
</style>
