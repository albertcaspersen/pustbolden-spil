<template>
  <div id="app">
    <img src="/src/pic/ChatGPT Image 6. nov. 2025, 09.32.13.png" alt="#" class="background">
    <div class="game-container">
      <div class="glass">
        <div class="playable-zone"></div>
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>

      <button @click="startGame" v-if="!gameStarted && !calibrating">Start Spil</button>

      <div v-if="calibrating">
        <!--<p>🤫 Kalibrerer... vær stille i 2 sekunder</p>-->
      </div>

      <div v-if="gameStarted">
        <!--<p>🎯 Blæs i mikrofonen for at holde bolden i luften!</p>-->
      </div>

      <div v-if="errorMsg" class="error-message">
        <p><strong>Fejl:</strong> {{ errorMsg }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import { gsap } from "gsap";

export default {
  data() {
    return {
      gameStarted: false,
      calibrating: false,
      errorMsg: null,

      // Lyd
      audioContext: null,
      analyser: null,
      highFrequencyEnergy: 0,
      rmsEnergy: 0,
      combinedEnergy: 0,
      baselineSlow: 0,
      baselineFast: 0,
      energyDelta: 0,
      prevEnergy: 0,
      relativeEnergy: 0,
      baselineReady: false,

      // Pust-detektion
      isBlowing: false,
      puffDetected: false,
      blowStartTime: 0,
      blowDuration: 0,
      minBlowDuration: 250,
      maxBlowDuration: 1000,
      puffCooldown: 500,
      lastPuffTime: 0,

      // Fysik
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.08,
      bounceDamping: 0.75,
      airResistance: 0.92,
      ballRadius: 15,

      // Parametre
      minHighHz: 2000,
      maxHighHz: 8000,
      energyThreshold: 5.5,
      smoothingFactor: 0.995,

      // ✅ Noise gate – filtrér alt under denne energi (juster 10–30)
      noiseGateThreshold: 30,

      // Animation
      animationId: null,
      ballSetterX: null,
      ballSetterY: null,
    };
  },

  mounted() {
    this.setBallPosition(50, 10);
  },

  beforeUnmount() {
    if (this.animationId) cancelAnimationFrame(this.animationId);
    if (this.audioContext && this.audioContext.state !== "closed")
      this.audioContext.close();
  },

  methods: {
    async startGame() {
      this.errorMsg = null;
      try {
        if (!this.audioContext) {
          this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }
        if (this.audioContext.state === "suspended") {
          await this.audioContext.resume();
        }

        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const source = this.audioContext.createMediaStreamSource(stream);

        const bandpass = this.audioContext.createBiquadFilter();
        bandpass.type = "bandpass";
        bandpass.frequency.value = 4000;
        bandpass.Q.value = 1.5;

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.smoothingTimeConstant = 0.15;

        source.connect(bandpass);
        bandpass.connect(this.analyser);

        // Kalibrer baggrundsstøj
        this.calibrating = true;
        await this.measureBaseline();
        this.calibrating = false;
        this.baselineReady = true;
        this.gameStarted = true;

        this.gameLoop();
      } catch (error) {
        console.error("Fejl:", error);
        this.errorMsg = "Fejl i mikrofontilladelse eller opsætning.";
      }
    },

    async measureBaseline() {
      return new Promise((resolve) => {
        const samples = [];
        const measure = () => {
          const freqEnergy = this.calculateHighFrequencyEnergy(this.analyser);
          const rms = this.calculateRmsEnergy(this.analyser);
          const combined = 0.6 * rms * 1000 + 0.4 * freqEnergy;

          if (!isNaN(combined) && combined > 0.1) {
            samples.push(combined);
          }
        };

        const interval = setInterval(measure, 100);

        setTimeout(() => {
          clearInterval(interval);
          if (samples.length === 0) {
            console.warn("⚠️ Ingen lyddata under kalibrering — sætter baseline til standardværdi.");
            this.baselineSlow = 20;
            this.baselineFast = 20;
          } else {
            const avg = samples.reduce((a, b) => a + b, 0) / samples.length;
            this.baselineSlow = avg;
            this.baselineFast = avg;
          }
          this.prevEnergy = this.baselineSlow;
          console.log("✅ Kalibrering færdig. Baseline:", this.baselineSlow.toFixed(1));
          resolve();
        }, 2500);
      });
    },

    calculateHighFrequencyEnergy(analyserNode) {
      const dataArray = new Uint8Array(analyserNode.frequencyBinCount);
      analyserNode.getByteFrequencyData(dataArray);
      const nyquist = this.audioContext.sampleRate / 2;
      const binSizeHz = nyquist / analyserNode.frequencyBinCount;
      const startBin = Math.floor(this.minHighHz / binSizeHz);
      const endBin = Math.ceil(this.maxHighHz / binSizeHz);

      let maxVal = 0;
      for (let i = startBin; i < endBin; i++) {
        if (dataArray[i] > maxVal) maxVal = dataArray[i];
      }
      return maxVal;
    },

    calculateRmsEnergy(analyserNode) {
      const dataArray = new Uint8Array(analyserNode.fftSize);
      analyserNode.getByteTimeDomainData(dataArray);
      let sumSquares = 0;
      for (let i = 0; i < dataArray.length; i++) {
        const v = (dataArray[i] - 128) / 128;
        sumSquares += v * v;
      }
      return Math.sqrt(sumSquares / dataArray.length);
    },

    setBallPosition(x, y) {
      if (!this.$refs.ballElement) return;
      if (!this.ballSetterX) {
        this.ballSetterX = gsap.quickSetter(this.$refs.ballElement, "left", "%");
        this.ballSetterY = gsap.quickSetter(this.$refs.ballElement, "bottom", "px");
      }
      const glassWidth = 100;
      const leftPercent = x - (this.ballRadius / glassWidth) * 100;
      this.ballSetterX(leftPercent);
      this.ballSetterY(y);
    },

    spawnNewBall() {
      this.ballX = 50;
      this.ballY = 10;
      this.velocityX = 0;
      this.velocityY = 0;
      this.setBallPosition(this.ballX, this.ballY);
    },

    detectBlow(now, energy) {

      // ✅ Noise Gate – ignorer alt under denne energi
      if (energy < this.noiseGateThreshold) {
        this.isBlowing = false;
        this.puffDetected = false;
        return false;
      }

      this.baselineSlow = this.baselineSlow * 0.995 + energy * 0.005;
      this.baselineFast = this.baselineFast * 0.8 + energy * 0.2;

      this.relativeEnergy = this.baselineFast - this.baselineSlow;
      this.energyDelta = energy - this.prevEnergy;
      this.prevEnergy = energy;

      const isStrongRise = this.energyDelta > 1.5;
      const isAboveThreshold = this.relativeEnergy > this.energyThreshold;

      if (!this.isBlowing && isStrongRise && isAboveThreshold) {
        this.isBlowing = true;
        this.blowStartTime = now;
      }

      if (this.isBlowing) {
        this.blowDuration = now - this.blowStartTime;
        if (this.blowDuration > this.maxBlowDuration || !isAboveThreshold) {
          this.isBlowing = false;
        }
      }

      if (
        this.isBlowing &&
        this.blowDuration > this.minBlowDuration &&
        now - this.lastPuffTime > this.puffCooldown
      ) {
        this.lastPuffTime = now;
        this.puffDetected = true;
        return true;
      }

      this.puffDetected = false;
      return false;
    },

    gameLoop() {
      const maxHeight = 400;
      const minY = -200;
      const viewportBottom = -250;

      const update = () => {
        const now = performance.now();
        const freqEnergy = this.calculateHighFrequencyEnergy(this.analyser);
        const rmsEnergy = this.calculateRmsEnergy(this.analyser);
        this.rmsEnergy = rmsEnergy;

        const combined = 0.6 * rmsEnergy * 1000 + 0.4 * freqEnergy;
        this.highFrequencyEnergy = freqEnergy;
        this.combinedEnergy = combined;

        const blowDetected = this.detectBlow(now, combined);

        if (blowDetected && this.ballY < maxHeight) {
          const force = (combined - this.baselineSlow) / 3;
          this.velocityY += force;
          this.velocityX += (Math.random() - 0.5) * 0.2;
        }

        this.velocityX *= this.airResistance;
        this.velocityY *= this.airResistance;
        this.velocityY -= this.gravity;
        this.ballX += this.velocityX;
        this.ballY += this.velocityY;

        if (this.ballY <= minY) {
          this.ballY = minY;
          this.velocityY = Math.abs(this.velocityY) * this.bounceDamping;
        }

        if (this.ballY > maxHeight) {
          this.ballY = maxHeight;
          this.velocityY = 0;
        }

        if (this.ballY < viewportBottom) this.spawnNewBall();

        this.setBallPosition(this.ballX, this.ballY);
        this.animationId = requestAnimationFrame(update);
      };
      update();
    },
  },
};
</script>

<style>
#app {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  font-family: sans-serif;
  text-align: center;
  position: relative;
}
.background {
  position: absolute;
  top: -2vh;
  left: -129vw;
  width: 66rem;
  height: 55rem;
  z-index: -4;
  overflow: hidden;
}
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  z-index: 4;
}
.glass {
  width: 100px;
  height: 200px;
  border-top: none;
  position: relative;
  border-radius: 0 0 10px 10px;
}
.playable-zone {
  position: absolute;
  bottom: -200px;
  left: 0;
  width: 100px;
  height: 600px;
  background-color: rgba(0, 255, 0, 0.2);
  border: 2px solid rgba(0, 255, 0, 0.5);
  border-radius: 5px;
  pointer-events: none;
  z-index: 0;
}
.ball {
  width: 60px;
  height: 60px;
  background: red;
  border-radius: 50%;
  position: absolute;
  bottom: 10px;
  z-index: 1;
}
.error-message {
  color: #D8000C;
  background: #FFD2D2;
  padding: 10px;
  border-radius: 5px;
  margin-top: 10px;
}
</style>
