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
        .then(async stream => {
          this.$refs.video.srcObject = stream;
          this.$refs.video.camonloadedmetadata = () => {
            this.$refs.video.play();
          };
          this.detection();
        })
        .catch(err => console.error(err));
    },

    async detection() {
      console.log("detection starts");
      const displaySize = {
        width: this.$refs.video.width,
        height: this.$refs.video.height
      };
      const labels = await this.loadLabels();
      faceapi.matchDimensions(this.$refs.canvas, displaySize);

      setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(
            this.$refs.video,
            new faceapi.TinyFaceDetectorOptions()
          )
          .withFaceLandmarks()
          .withFaceExpressions()
          .withAgeAndGender()
          .withFaceDescriptors();
        const resizedDetections = faceapi.resizeResults(
          detections,
          displaySize
        );
        const faceMatcher = new faceapi.FaceMatcher(labels, 0.6);
        const results = resizedDetections.map(d => {
          return faceMatcher.findBestMatch(d.descriptor);
        });
        this.$refs.canvas.getContext("2d").clearRect(0, 0, 640, 480);
        faceapi.draw.drawDetections(this.$refs.canvas, resizedDetections);
        faceapi.draw.drawFaceExpressions(this.$refs.canvas, resizedDetections);
        results.forEach((result, index) => {
          const box = resizedDetections[index].detection.box;
          const { label, distance } = result;
          new faceapi.draw.DrawTextField(
            [`${label} (${parseInt(distance, 10)})`],
            box.bottomRight
          ).draw(this.$refs.canvas);
        });
      }, 200);
    },

    loadLabels() {
      const labels = ["guilherme", "jojo", "gustavo"];
      return Promise.all(
        labels.map(async label => {
          const descriptions = [];
          for (let i = 1; i < 3; i++) {
            const img = await faceapi.fetchImage(`/labels/${label}/${i}.jpeg`);
            const detections = await faceapi
              .detectSingleFace(img)
              .withFaceLandmarks()
              .withFaceDescriptor();
            descriptions.push(detections.descriptor);
          }
          return new faceapi.LabeledFaceDescriptors(label, descriptions);
        })
      );
    },

    loadfaceapi() {
      Promise.all([
        faceapi.nets.tinyFaceDetector.loadFromUri("/models"),
        faceapi.nets.faceLandmark68Net.loadFromUri("/models"),
        faceapi.nets.faceRecognitionNet.loadFromUri("/models"),
        faceapi.nets.faceExpressionNet.loadFromUri("/models"),
        faceapi.nets.ageGenderNet.loadFromUri("/models"),
        faceapi.nets.ssdMobilenetv1.loadFromUri("/models")
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
