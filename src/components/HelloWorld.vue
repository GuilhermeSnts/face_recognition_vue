<template>
  <div class="hello">
    <video
      ref="video"
      width="400"
      height="400"
      playsinline
      muted
      autoplay
    ></video>
    <canvas ref="canvas" width="400" height="400"></canvas>
  </div>
</template>

<script>
import * as faceapi from "face-api.js";
export default {
  name: "HelloWorld",

  props: {
    msg: String
  },

  data: () => ({
    video: {},
    canvas: {},
    captures: []
  }),

  methods: {
    startVideo() {
      navigator.mediaDevices
        .getUserMedia({ video: true })
        .then(stream => {
          this.$refs.video.srcObject = stream;
          this.$refs.video.camonloadedmetadata = () => {
            this.$refs.video.play();
          };
          this.detection();
        })
        .catch(err => console.error(err));
    },

    detection() {
      console.log("detection starts");
      const displaySize = {
        width: this.$refs.video.width,
        height: this.$refs.video.height
      };
      faceapi.matchDimensions(this.$refs.canvas, displaySize);

      setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(
            this.$refs.video,
            new faceapi.TinyFaceDetectorOptions()
          )
          .withFaceExpressions();
        const resizedDetections = faceapi.resizeResults(
          detections,
          displaySize
        );
        this.$refs.canvas.getContext("2d").clearRect(0, 0, 640, 480);
        faceapi.draw.drawDetections(this.$refs.canvas, resizedDetections);
        faceapi.draw.drawFaceExpressions(this.$refs.canvas, resizedDetections);
      }, 200);
    },

    loadfaceapi() {
      Promise.all([
        faceapi.nets.tinyFaceDetector.loadFromUri("/models"),
        faceapi.nets.faceLandmark68Net.loadFromUri("/models"),
        faceapi.nets.faceRecognitionNet.loadFromUri("/models"),
        faceapi.nets.faceExpressionNet.loadFromUri("/models"),
        faceapi.nets.ageGenderNet.loadFromUri("/models")
      ])
        .then(this.startVideo())
        .catch(err => console.error(err));
    }
  },
  mounted() {
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      this.loadfaceapi();
    }
  }
};
</script>

<style scoped>
canvas,
video {
  position: absolute;
  top: 0;
  left: 0;
}
</style>
