<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <div class="playable-zone"></div>
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>

      <button @click="startGame" v-if="!gameStarted && !calibrating">Start Spil</button>

      <div v-if="calibrating">
        <p>🤫 Kalibrerer... vær stille i 2 sekunder</p>
      </div>

      <div v-if="gameStarted">
        <p>🎯 Blæs i mikrofonen for at holde bolden i luften!</p>
        <div class="debug-panel">
          <p><strong>Debug Info</strong></p>
          <p>Lydenergi: {{ highFrequencyEnergy.toFixed(1) }}</p>
          <p>Baseline: {{ baselineEnergy.toFixed(1) }}</p>
          <p>Relativ Energi: 
            <span :class="{ positive: relativeEnergy > 0, negative: relativeEnergy < 0 }">
              {{ relativeEnergy.toFixed(1) }}
            </span>
          </p>
          <p>Varighed over threshold: {{ blowDuration }} ms</p>
          <p>Pust aktivt: <strong>{{ isBlowing ? "💨" : "..." }}</strong></p>
          <p>Registreret pust: <strong>{{ puffDetected ? "✅" : "—" }}</strong></p>
        </div>
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
      // Tilstand
      gameStarted: false,
      calibrating: false,
      errorMsg: null,

      // Lyd
      audioContext: null,
      analyser: null,
      highFrequencyEnergy: 0,
      baselineEnergy: 0,
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
      airResistance: 0.97,
      ballRadius: 15,
      ballEjected: false,

      // Parametre
      minHighHz: 2000,
      maxHighHz: 8000,
      relativeEnergyThreshold: 6,
      lowerEnergyThreshold: 2.5,
      smoothingFactor: 0.995,

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

        // Bandpass – fokuser på pustefrekvenser (2–8 kHz)
        const bandpass = this.audioContext.createBiquadFilter();
        bandpass.type = "bandpass";
        bandpass.frequency.value = 4000;
        bandpass.Q.value = 1.5;

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.smoothingTimeConstant = 0.2;

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
        const measure = () => samples.push(this.calculateHighFrequencyEnergy(this.analyser));
        const interval = setInterval(measure, 100);
        setTimeout(() => {
          clearInterval(interval);
          this.baselineEnergy = samples.reduce((a, b) => a + b, 0) / samples.length;
          resolve();
        }, 2000);
      });
    },

    calculateHighFrequencyEnergy(analyserNode) {
      const dataArray = new Uint8Array(analyserNode.frequencyBinCount);
      analyserNode.getByteFrequencyData(dataArray);
      const nyquist = this.audioContext.sampleRate / 2;
      const binSizeHz = nyquist / analyserNode.frequencyBinCount;
      const startBin = Math.floor(this.minHighHz / binSizeHz);
      const endBin = Math.ceil(this.maxHighHz / binSizeHz);

      // Brug max frem for at fange korte pust
      let maxVal = 0;
      for (let i = startBin; i < endBin; i++) {
        if (dataArray[i] > maxVal) maxVal = dataArray[i];
      }
      return maxVal;
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
      this.ballEjected = false;
      this.setBallPosition(this.ballX, this.ballY);
    },

    detectBlow(now, energy) {
      // Opdater baseline kun når der ikke pustes
      if (!this.isBlowing) {
        this.baselineEnergy =
          this.smoothingFactor * this.baselineEnergy + (1 - this.smoothingFactor) * energy;
      }

      // Beregn relativ energi
      this.relativeEnergy = Math.max(0, energy - this.baselineEnergy);

      // Hysterese thresholds
      const upperThreshold = this.relativeEnergyThreshold;
      const lowerThreshold = this.lowerEnergyThreshold;

      if (this.isBlowing) {
        if (this.relativeEnergy < lowerThreshold || this.blowDuration > this.maxBlowDuration) {
          this.isBlowing = false;
          this.blowDuration = 0;
        } else {
          this.blowDuration = now - this.blowStartTime;
        }
      } else {
        if (this.relativeEnergy > upperThreshold) {
          this.isBlowing = true;
          this.blowStartTime = now;
          this.blowDuration = 0;
        }
      }

      // Kræv min. varighed før pust registreres
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
      const glassWidth = 100;
      const glassHeight = 200;
      const minY = 10;
      const maxHeight = 250;
      const minX = (this.ballRadius / glassWidth) * 100;
      const maxX = 100 - minX;
      const viewportBottom = -150;

      const update = () => {
        const now = performance.now();
        const newEnergy = this.calculateHighFrequencyEnergy(this.analyser);

        // Glat energien
        this.highFrequencyEnergy = parseFloat((0.85 * this.highFrequencyEnergy + 0.15 * newEnergy).toFixed(1));

        const blowDetected = this.detectBlow(now, this.highFrequencyEnergy);

        if (blowDetected && this.ballY < maxHeight) {
          const force = (this.highFrequencyEnergy - this.baselineEnergy) / 40;
          this.velocityY += force;
          this.velocityX += (Math.random() - 0.5) * 0.2;
        }

        // Fysik
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
}
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}
.glass {
  width: 100px;
  height: 200px;
  border: 5px solid #ccc;
  border-top: none;
  position: relative;
  border-radius: 0 0 10px 10px;
}
.ball {
  width: 30px;
  height: 30px;
  background: red;
  border-radius: 50%;
  position: absolute;
  bottom: 10px;
}
.debug-panel {
  margin-top: 10px;
  padding: 8px;
  font-family: monospace;
  border: 1px solid #ccc;
  background: #f9f9f9;
  border-radius: 6px;
  text-align: left;
}
.debug-panel .positive {
  color: green;
}
.debug-panel .negative {
  color: #c00;
}
.error-message {
  color: #D8000C;
  background: #FFD2D2;
  padding: 10px;
  border-radius: 5px;
  margin-top: 10px;
}
</style>
