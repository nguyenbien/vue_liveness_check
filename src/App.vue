<template>
  <div>
    <div class="column">
      <h :style="{color: !isBlinkEye ? 'black' : 'red', padding: '10px'}">blink_eye</h>
      <h :style="{color: !isSmile ? 'black' : 'red', padding: '10px'}">smile</h>
      <h :style="{color: !isTurnRight ? 'black' : 'red', padding: '10px'}">turn_right</h>
      <h :style="{color: !isTurnLeft ? 'black' : 'red', padding: '10px'}">turn_left</h>
    </div>
    <div class="container" style="position: relative;">
      <video ref="input_video" style="position: abso left: 0px; top: 0px;" autoplay playsinline width=320 height=247></video>
      <canvas
        class="image_canvas"
        ref="image_canvas"
        style="position: absolute; left: 0px; top: 0px;"
        width=320
        height=247
      ></canvas>

      <canvas
        class="output_canvas"
        ref="output_canvas"
        style="position: absolute; left: 0px; top: 0px;"
        width=320
        height=247
      ></canvas>
      <p :style="{color: !isFake ? 'green' : 'red', position: 'absolute', left: fakeLeft, top: fakeTop}"> {{faceName}} </p>
    </div>
    <div class="column">
      <img :src="eye_blink" width="150" height="116">
      <img :src="mouth_smile" width="150" height="116">
      <img :src="turn_right" width="150" height="116">
      <img :src="turn_left" width="150" height="116">
    </div>
    <div>
      <video ref="recoder_video" controls playsinline width=320 height=247></video>
    </div>
  </div>
</template>
<script>
import { Camera } from "@mediapipe/camera_utils";
import vision from "@mediapipe/tasks-vision";
const { FaceLandmarker, FilesetResolver, DrawingUtils, ImageClassifier} = vision;

export default {
  data: function() {
    return {
      isFake:false,
      fakeLeft:'0px',
      fakeTop:'0px',
      faceName:'REAL',
      number: null,
      ctx: null,
      image_ctx:null,
      imageCanvas:null,
      width: 320,
      height: 247,
      listDetection:["blink_eye", "smile", "turn_right", "turn_left"],
      indexDetection:0,
      isBlinkEye:false,
      isSmile:false,
      isTurnRight:false,
      isTurnLeft:false,
      googleCamera:null,
      totalSpoof:0,
      recorder:null,
      eye_blink:null,
      mouth_smile:null,
      turn_right:null,
      turn_left:null,
      isStartRecoder:false,
      isEkycDone:false,
      isStartEkyc:false,
      timerEkycTimeout: null,
      ekycTimeout:null,
      isNoFace:true,
      currtimeVideoEkyc:Date.now()
    };
  },
  computed: {
    inputVideo() {
      return this.$refs.input_video
    },
    recoderVideo(){
      return this.$refs.recoder_video
    }
  },
  async mounted() {
    this.ctx = this.$refs.output_canvas.getContext("2d")
    this.image_ctx = this.$refs.image_canvas.getContext("2d")
    this.imageCanvas = this.$refs.image_canvas
    await this.init()
  },
  methods: {
    async init() {
      this.totalSpoof = 0
      const filesetResolver = await FilesetResolver.forVisionTasks(
        "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm"
      );

      const faceLandmarker = await FaceLandmarker.createFromOptions(filesetResolver, {
          baseOptions: {
            modelAssetPath: `https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task`,
            delegate: "GPU"
          },
          outputFaceBlendshapes: true,
          runningMode:"VIDEO",
          numFaces: 1
        });

      const imageClassifier = await ImageClassifier.createFromOptions(
        filesetResolver,
        {
          baseOptions: {
            modelAssetPath: 'app/shared/models/model.tflite'
          },
          runningMode:"VIDEO",
          maxResults: 1
        });


      const stream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: false,
      })

      this.inputVideo.srcObject = stream

      if (!MediaRecorder.isTypeSupported('video/webm')) { // <2>
        console.warn('video/webm is not supported')
      }

      this.recorder = new MediaRecorder(stream, {
        mimeType: 'video/webm'
      });

      this.recorder.addEventListener('dataavailable', event => {
        if(this.isEkycDone){
          this.recoderVideo.src = URL.createObjectURL(event.data)
          console.log("Video data")
          console.log(this.recoderVideo.src)
        }
        
      })

      this.googleCamera = new Camera(this.inputVideo, {
        onFrame: async () => {
          let nowInMs = Date.now()
          if(this.currtimeVideoEkyc != nowInMs){
            this.currtimeVideoEkyc = nowInMs
            this.image_ctx.drawImage(this.inputVideo, 0, 0, this.width, this.height);
            var imgAsDataURL = this.imageCanvas.toDataURL("image/png");
            const results = faceLandmarker.detectForVideo(this.inputVideo, nowInMs)
            if(results){
              if(!this.isStartRecoder){
                this.isStartRecoder = true
                this.recorder.start()
              }
              
              this.onResults(results)
              if(results.faceBlendshapes && results.faceBlendshapes.length > 0){
                if(this.isStartEkyc){
                  this.checkEkycInfo(results.faceBlendshapes[0].categories, imgAsDataURL)
                }
              }

              if(results.faceLandmarks && results.faceLandmarks.length > 0){
                this.isNoFace = false
                const landmarks = results.faceLandmarks[0]

                const minX = this.findMinX(landmarks) * this.width
                const maxX = this.findMaxX(landmarks) * this.width
                const minY = this.findMinY(landmarks) * this.height
                const maxY = this.findMaxY(landmarks) * this.height
                const hBox = maxY - minY
                const wBox = maxX - minX
                const drawingUtils = new DrawingUtils(this.ctx);

                let categoryName = "live"
                const box = this.boundingBoxFaceDetection(minX, minY, maxX, maxY)
                const imageClassifierResult = imageClassifier.classifyForVideo(this.inputVideo, nowInMs, {regionOfInterest:box})
                if(imageClassifierResult.classifications && imageClassifierResult.classifications.length > 0 
                && imageClassifierResult.classifications[0].categories && imageClassifierResult.classifications[0].categories.length > 0){
                  categoryName = imageClassifierResult.classifications[0].categories[0].categoryName
                }

                if(categoryName.localeCompare("live") != 0){
                  console.log("SPOOF FACE")
                  this.isFake = true
                  this.fakeLeft = (maxX + 10)+'px'
                  this.fakeTop = minY+'px'
                  this.faceName = 'FAKE'
                  drawingUtils.drawBoundingBox({
                    originX:minX,
                    originY:minY,
                    width:wBox,
                    height:hBox
                  }, {
                    color:"#ff0000",
                    lineWidth:2,
                    fillColor:"#00000000"
                  })
                  // this.totalSpoof += 1
                  // if(this.totalSpoof >= 2){
                  //   console.log("No Face")
                  //   // this.onStopEkyc()
                  //   // this.showDialogNoFace()
                  // }
                }else{
                  this.isFake = false
                  this.fakeLeft = (maxX + 10)+'px'
                  this.fakeTop = minY+'px'
                  this.faceName = 'REAL'
                  drawingUtils.drawBoundingBox({
                    originX:minX,
                    originY:minY,
                    width:wBox,
                    height:hBox
                    }, {
                      color:"#30FF30",
                      lineWidth:2,
                      fillColor:"#00000000"
                  })
                }

              } 
            }
          }
          
        },
      });

      this.googleCamera.start()
      this.delayCheckEkyc()
      this.startTimerEkycTimeout()
    },
    startTimerEkycTimeout(){
      this.timerEkycTimeout = setInterval(this.showEkycTimeout, 1000000000)
    },
    stopTimerEkycTimeout(){
      clearInterval(this.timerEkycTimeout)
      this.timerEkycTimeout = null
    },
    showEkycTimeout( )
    {
      if(this.isNoFace){
        this.onStopEkyc()
        this.showDialogNoFace()
      }else{
        this.onStopEkyc()
        this.showDialogLostFace()
      }
    },
    onResults(results) {
      this.width = 320
      this.height = 247
      this.ctx.save()
      this.ctx.clearRect(0, 0, this.width, this.height)
      this.findLandmarks(results)
      this.ctx.restore()
    },
    delayCheckEkyc(){
      this.isStartEkyc = false
      this.ekycTimeout = setTimeout(this.startCheckEkyc, 300)
    },
    cancelCheckEkyc(){
      clearTimeout(this.ekycTimeout)
      this.ekycTimeout = null
      this.isStartEkyc = false
    },
    startCheckEkyc(){
      this.isStartEkyc = true
    },
    checkEkycInfo(categories, imgAsDataURL){
      if(this.indexDetection == 0){
        const i_e_blink_left = categories.findIndex(x => x.categoryName ==="eyeBlinkLeft")
        const i_e_blink_right = categories.findIndex(x => x.categoryName ==="eyeBlinkRight")
        const eyeBlinkLeft = categories[i_e_blink_left].score
        const eyeBlinkRight = categories[i_e_blink_right].score
        const eyeBlink = (eyeBlinkLeft + eyeBlinkRight)/2
        const isBlink = this.checkEyeBlink(eyeBlink)
        if(isBlink){
          this.indexDetection = 1
          this.isBlinkEye = true
          this.eye_blink = imgAsDataURL
          this.delayCheckEkyc()
          this.stopTimerEkycTimeout()
          this.startTimerEkycTimeout()
        }
    
      }else if(this.indexDetection == 1){
        const i_m_smile_left = categories.findIndex(x => x.categoryName ==="mouthSmileLeft")
        const i_m_smile_right = categories.findIndex(x => x.categoryName ==="mouthSmileRight")
        const mouthSmileLeft = categories[i_m_smile_left].score
        const mouthSmileRight = categories[i_m_smile_right].score
        const mouthSmile = (mouthSmileLeft + mouthSmileRight)/2
        const isSmile = this.checkMouthSmile(mouthSmile)
        if(isSmile){
          this.indexDetection = 2
          this.isSmile = true
          this.mouth_smile = imgAsDataURL
          this.delayCheckEkyc()
          this.stopTimerEkycTimeout()
          this.startTimerEkycTimeout()
        }
      }else if(this.indexDetection == 2){
        const i_look_right = categories.findIndex(x => x.categoryName ==="eyeLookInRight");
        const i_look_left = categories.findIndex(x => x.categoryName ==="eyeLookOutLeft");
        const browDownRight = categories[i_look_right].score
        const browDownLeft = categories[i_look_left].score
        const isBrowRight= this.checkBrowDownRight((browDownRight + browDownLeft) /2)
        if(isBrowRight){
          this.indexDetection = 3
          this.isTurnRight = true
          this.turn_right = imgAsDataURL
          this.delayCheckEkyc()
          this.stopTimerEkycTimeout()
          this.startTimerEkycTimeout()
        }
      }else if(this.indexDetection == 3){
        const i_look_left = categories.findIndex(x => x.categoryName ==="eyeLookInLeft");
        const i_look_right = categories.findIndex(x => x.categoryName ==="eyeLookOutRight");
        const browDownLeft = categories[i_look_left].score
        const browDownRight = categories[i_look_right].score
        const isBrowLeft = this.checkBrowDownLeft((browDownLeft + browDownRight)/2)
        if(isBrowLeft){
          this.indexDetection = -1
          this.isTurnLeft = true
          this.turn_left = imgAsDataURL
          this.startCheckEkyc = false;
          this.isEkycDone = true
          this.recorder.stop()
          this.isStartRecoder = false
          this.googleCamera.stop()
          this.stopTimerEkycTimeout()
        }
      }
    },
    checkEyeBlink(eyeBlink){
      if(eyeBlink >= 0.7){
        return true
      }

      return false
    },
    checkMouthSmile(mouthSmile){
      if(mouthSmile >= 0.6){
        return true
      }

      return false
    },
    checkBrowDownLeft(browDownLeft){
      if(browDownLeft >= 0.8){
        return true
      }

      return false
    },
    checkBrowDownRight(browDownRight){
      if(browDownRight >= 0.8){
        return true
      }

      return false
    },
    findLandmarks(results) {
      // const drawingUtils = new DrawingUtils(this.ctx);
      // if(results.faceLandmarks && results.faceLandmarks.length > 0){
      //   for (const landmarks of results.faceLandmarks) {
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_TESSELATION,
      //       { color: "#C0C0C070", lineWidth: 1 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_RIGHT_EYE,
      //       { color: "#FF3030" , lineWidth: 2}
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_RIGHT_EYEBROW,
      //       { color: "#FF3030", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_LEFT_EYE,
      //       { color: "#30FF30", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_LEFT_EYEBROW,
      //       { color: "#30FF30", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_FACE_OVAL,
      //       { color: "#E0E0E0", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_LIPS,
      //       { color: "#E0E0E0", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_RIGHT_IRIS,
      //       { color: "#FF3030", lineWidth: 2 }
      //     );
      //     drawingUtils.drawConnectors(
      //       landmarks,
      //       FaceLandmarker.FACE_LANDMARKS_LEFT_IRIS,
      //       { color: "#30FF30", lineWidth: 2 }
      //     );
      //   }
      // }
    },
    findMaxY(faceLandmarks){
      let max = -Infinity
      let key = 0

      faceLandmarks.forEach(function (v, k) { 
          if (max < v.y) { 
              max = v.y; 
              key = k; 
          }
      });

      return faceLandmarks[key].y
    },
    findMinY(faceLandmarks){
      let max = Infinity
      let key = 0

      faceLandmarks.forEach(function (v, k) { 
          if (max > v.y) { 
              max = v.y; 
              key = k; 
          }
      });

      return faceLandmarks[key].y
    },
    findMaxX(faceLandmarks){
      let max = -Infinity
      let key = 0

      faceLandmarks.forEach(function (v, k) { 
          if (max < v.x) { 
              max = v.x; 
              key = k; 
          }
      });

      return faceLandmarks[key].x
    },
    findMinX(faceLandmarks){
      let max = Infinity
      let key = 0

      faceLandmarks.forEach(function (v, k) { 
          if (max > v.x) { 
              max = v.x; 
              key = k; 
          }
      });

      console.log(faceLandmarks[key])

      return faceLandmarks[key].x
    },
    boundingBoxFaceDetection(x, y, right, bottom){
      const box_width = right - x
      const box_height = bottom - y
      const shift_x = 0
      const shift_y = 0
      const scale = Math.min(2.7, Math.min((this.width - 1)/box_width, (this.height - 1)/box_height))


      const box_center_x = box_width / 2 + x
      const  box_center_y = box_height / 2 + y
      const max = Math.max(box_width, box_height)
      const new_width = parseInt(max * scale)
      const new_height = parseInt(max * scale)
      var left_top_x = box_center_x - new_width / 2 + shift_x
      var left_top_y = box_center_y - new_height / 2 + shift_y
      var right_bottom_x = box_center_x + new_width / 2 + shift_x
      var right_bottom_y = box_center_y + new_height / 2 + shift_y

      
      if (right_bottom_x >= this.width) {
          const s = right_bottom_x - this.width + 1
          left_top_x -= s
          right_bottom_x -= s
      }
      if (right_bottom_y >= this.height) {
          const s = right_bottom_y - this.height + 1
          left_top_y -= s
          right_bottom_y -= s
      }

      if (left_top_x < 0) {
          right_bottom_x -= left_top_x
          left_top_x = 0
      }
      if (left_top_y < 0) {
          right_bottom_y -= left_top_y
          left_top_y = 0
      }

      const n_right = left_top_x + new_width
      const n_bottom = left_top_y + new_height
      return {
        left:Math.min(left_top_x/this.width, 1),
        top:Math.min(left_top_y/this.height, 1),
        right:Math.min(n_right/this.width, 1),
        bottom:Math.min(n_bottom/this.height, 1)
      }
    },
    onResumeEkyc(){
      this.totalSpoof = 0
      this.ctx.save()
      this.ctx.clearRect(0, 0, this.width, this.height)
      this.ctx.restore()
      this.image_ctx.save();
      this.image_ctx.clearRect(0, 0, this.width, this.height)
      this.image_ctx.restore()
      this.isEkycDone = false
      this.eye_blink = null
      this.mouth_smile = null
      this.turn_left = null
      this.turn_right = null
      this.indexDetection = 0
      this.isBlinkEye = false
      this.isSmile = false
      this.isTurnRight = false
      this.isTurnLeft = false
      this.isStartEkyc = false
      this.isNoFace = true
      this.googleCamera.start()
      this.recorder.start()
      this.delayCheckEkyc()
      this.startTimerEkycTimeout()
    },
    onStopEkyc(){
      // this.isNoFace = true
      // this.isEkycDone = false
      // this.isStartRecoder = false
      this.recorder.stop()
      this.googleCamera.stop()
      this.cancelCheckEkyc()
      this.stopTimerEkycTimeout()
    },
    showDialogLostFace(){
      if (confirm("Đã quá thời gian thực hiện thao tác. Vui lòng thực hiện lại.")) {
        this.onResumeEkyc()
      } else {
        
      }
    },
    showDialogNoFace(){
      if (confirm("Vui lòng đưa khuôn mặt bạn vào khung hình.")) {
        this.onResumeEkyc()
      } else {
        
      }
    },
  },
};
</script>

<style scoped>


/* .input_video {
  clear: both;
  display: block;
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
}

.output_canvas {
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
}

.image_canvas {
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
} */

video {
  clear: both;
  display: block;
  transform: rotateY(0deg);
  -webkit-transform: rotateY(0deg);
  -moz-transform: rotateY(0deg);
}

.output_canvas {
  transform: rotateY(0deg);
  -webkit-transform: rotateY(0deg);
  -moz-transform: rotateY(0deg);
}

.image_canvas {
  transform: rotateY(0deg);
  -webkit-transform: rotateY(0deg);
  -moz-transform: rotateY(0deg);
}

.canvas {
  z-index: 1;
  position: absolute;
  pointer-events: none;
} 

</style>