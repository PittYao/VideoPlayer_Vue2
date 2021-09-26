<template>
  <video ref="videoElem" autoplay></video>
</template>

<script>
import { get, post } from "@/utils/request.js";

export default {
  name: "VideoPlayer",
  props: {
    rtspUrl: String,
    disableAudio: {
      type: Boolean,
      default: false,
    },
    play: {
      type: Boolean,
      default: true,
    },
    muted: {
      type: Boolean,
      default: false,
    },
    full: {
      type: Boolean,
      default: false,
    },
  },
  mounted() {
    // 起始步骤
    if (this.rtspUrl === "" || this.rtspUrl === undefined) {
      return;
    }
    // 自动播放
    if (this.play) {
      this.playVideo();
    }
     // 全屏监听
      document.addEventListener("fullscreenchange", () => {
        if (!this.checkFull()) {
          console.log("退出全屏");
          this.$emit("update:full", false);
        } else {
          console.log("进入全屏");
        }
      });
  },
  data() {
    return {
      pcObj: {
        pc: null,
        sendChannel: null,
        stream: null,
        suuid: 0,
        rstpUrl: "rtsp://admin:cebon61332433@192.168.99.215",
        disableAudio: false,
        // 接口地址
        registerUrl: "/stream/register",
        receiverUrl: "/stream/receiver/",
        codecUrl: "/stream/codec/",
      },
    };
  },
  watch: {
    play(newPlay) {
      newPlay === true ? this.playVideo() : this.pcClear();
    },
    muted(newMuted) {
      console.log(newMuted);
      newMuted === true
        ? (this.$refs.videoElem.muted = "muted")
        : (this.$refs.videoElem.muted = "");
    },
    full(newFull) {
      if (newFull === true) {
        this.fullScreen();
      }
    },
  },
  methods: {
    // 检测全屏元素
    checkFull() {
      var isFull =
        document.fullscreenElement ||
        document.mozFullScreenElement ||
        document.webkitFullscreenElement;
      //to fix : false || undefined == undefined
      if (isFull === undefined) isFull = false;
      return isFull;
    },
    // 全屏
    fullScreen() {
      const videoElemValue = this.$refs.videoElem;

      // 适配各浏览器
      if (videoElemValue.requestFullscreen) {
        videoElemValue.requestFullscreen();
      } else if (videoElemValue.webkitRequestFullscreen) {
        videoElemValue.webkitRequestFullscreen();
      } else if (videoElemValue.mozRequestFullScreen) {
        videoElemValue.mozRequestFullScreen();
      }
    },
    // 清空pc
    pcClear() {
      this.$refs.videoElem.srcObject = null;
      this.pcObj.pc = null;
      this.pcObj.sendChannel = null;
      this.pcObj.stream = null;
    },
    // 重置pc
    pcReload() {
      this.pcClear();

      let stream = new MediaStream();
      this.pcObj.stream = stream;

      let config = {
        iceServers: [
          {
            urls: ["stun:stun.l.google.com:19302"],
          },
        ],
      };

      this.pcObj.pc = new RTCPeerConnection(config);
      this.pcObj.pc.onnegotiationneeded = this.handleNegotiationNeededEvent;

      this.pcObj.pc.ontrack = (event) => {
        this.pcObj.stream.addTrack(event.track);
        this.$refs.videoElem.srcObject = this.pcObj.stream;
        console.log(event.streams.length + " track is delivered");
        console.log(this.$refs.videoElem);
      };

      this.pcObj.pc.oniceconnectionstatechange = (e) => {
        console.log(e);
        if (this.pcObj.pc !== null) {
          console.log(this.pcObj.pc.iceConnectionState);
        }
      };
    },
    async handleNegotiationNeededEvent() {
      let offer = await this.pcObj.pc.createOffer();
      await this.pcObj.pc.setLocalDescription(offer);
      this.getRemoteSdp();
    },
    async getRemoteSdp() {
      try {
        const sdpData = {
          suuid: this.pcObj.suuid,
          data: btoa(this.pcObj.pc.localDescription.sdp),
        };

        const result = await post(
          this.pcObj.receiverUrl + this.pcObj.suuid,
          sdpData
        );

        try {
          this.pcObj.pc.setRemoteDescription(
            new RTCSessionDescription({
              type: "answer",
              sdp: Buffer.from(result, "base64"),
            })
          );
        } catch (e) {
          console.warn(e);
        }
      } catch (error) {
        console.log("receiver请求失败", error);
      }
    },
    async getCodecInfo() {
      let result = await get(this.pcObj.codecUrl + this.pcObj.suuid);

      result.forEach((value) => {
        this.pcObj.pc.addTransceiver(value.Type, {
          direction: "sendrecv",
        });
      });

      this.pcObj.sendChannel = this.pcObj.pc.createDataChannel("foo");
      this.pcObj.sendChannel.onclose = () =>
        console.log("sendChannel has closed");
      this.pcObj.sendChannel.onopen = () => {
        console.log("sendChannel has opened");
        this.pcObj.sendChannel.send("ping");
        let interval = setInterval(() => {
          if (this.pcObj.sendChannel === null) {
            clearInterval(interval);
          } else if (this.pcObj.sendChannel.readyState === "open") {
            this.pcObj.sendChannel.send("ping");
          }
        }, 1000);
      };

      this.pcObj.sendChannel.onmessage = (e) => console.log(e);
    },
    // 播放视频
    async playVideo() {
      // 重置
      this.pcReload();
      // 清空
      console.log("pc状态：", this.pcObj.pc.connectionState);
      if (this.pcObj.pc.connectionState === "connected") {
        // 连接状态断开
        this.pcObj.pc.close();
        console.log("pc关闭后建立状态：", this.pcObj.pc.connectionState);
      }

      console.log("rtspUrl:", this.pcObj.rstpUrl);

      let send_data = {
        rtspUrl: this.pcObj.rstpUrl,
        disableAudio: this.pcObj.disableAudio,
      };

      const result = await post(this.pcObj.registerUrl, send_data);

      try {
        if (result.code === 200) {
          this.pcObj.suuid = result.data;
          // 请求编解码
          this.getCodecInfo();
        }
      } catch (error) {
        console.warn("register 注册失败", error);
      }
    },
  },
};
</script>

<style lang="scss" scoped>
video {
  width: 600px;
}
</style>