<template>
  <div id="app" class="container">
    <h1>Pust bolden ud af glasset</h1>
    <div class="glass">
      <div class="ball" :style="{ transform: `translateY(-${ballPosition}px)` }"></div>
    </div>
    <button @click="startMic" v-if="!micStarted">Start Mikrofon</button>
    <p v-if="micStarted && !isBlowing">Pust i mikrofonen for at løfte bolden.</p>
    <p v-if="isBlowing">Bliv ved!</p>
    <p v-if="error">{{ error }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      ballPosition: 0,
      micStarted: false,
      isBlowing: false,
      error: null,
      audioContext: null,
      analyser: null,
      dataArray: null,
      animationFrameId: null,
    };
  },
  methods: {
    startMic() {
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices.getUserMedia({ audio: true })
          .then(stream => {
            this.micStarted = true;
            this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
            this.analyser = this.audioContext.createAnalyser();
            const source = this.audioContext.createMediaStreamSource(stream);
            source.connect(this.analyser);
            this.analyser.fftSize = 256;
            const bufferLength = this.analyser.frequencyBinCount;
            this.dataArray = new Uint8Array(bufferLength);
            this.detectBlow();
          })
          .catch(err => {
            this.error = "Fejl ved adgang til mikrofon: " + err.message;
          });
      } else {
        this.error = "Din browser understøtter ikke mikrofon-adgang.";
      }
    },
    detectBlow() {
      this.animationFrameId = requestAnimationFrame(this.detectBlow);
      this.analyser.getByteFrequencyData(this.dataArray);

      // Simple gennemsnitslydstyrke
      let sum = 0;
      for (let i = 0; i < this.dataArray.length; i++) {
        sum += this.dataArray[i];
      }
      let averageVolume = sum / this.dataArray.length;

      // Tjek for bredbåndsstøj (karakteristisk for pust)
      // Vi tjekker, om der er betydelig energi i flere frekvensbånd
      const lowerHalf = this.dataArray.slice(0, this.dataArray.length / 2);
      const upperHalf = this.dataArray.slice(this.dataArray.length / 2);
      const lowerAvg = lowerHalf.reduce((a, b) => a + b, 0) / lowerHalf.length;
      const upperAvg = upperHalf.reduce((a, b) => a + b, 0) / upperHalf.length;

      // Juster disse tærskler efter behov for at tilpasse følsomheden
      const volumeThreshold = 20; // Samlet lydstyrketærskel
      const noiseThreshold = 1.5; // Forholdet mellem høje og lave frekvenser

      if (averageVolume > volumeThreshold && upperAvg > lowerAvg * noiseThreshold) {
        this.isBlowing = true;
        this.ballPosition = Math.min(this.ballPosition + averageVolume / 10, 300); // 300 er glassets højde
      } else {
        this.isBlowing = false;
        this.ballPosition = Math.max(this.ballPosition - 5, 0);
      }
    },
  },
  beforeUnmount() {
    if (this.animationFrameId) {
      cancelAnimationFrame(this.animationFrameId);
    }
    if (this.audioContext) {
      this.audioContext.close();
    }
  },
};
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  font-family: sans-serif;
}

.glass {
  width: 150px;
  height: 300px;
  border: 5px solid black;
  border-top: none;
  display: flex;
  flex-direction: column-reverse;
  align-items: center;
}

.ball {
  width: 50px;
  height: 50px;
  background-color: red;
  border-radius: 50%;
  transition: transform 0.1s linear;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>