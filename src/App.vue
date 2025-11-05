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
      baselineReady: false,
      prevEnergy: 0,

      // Pust-detektion
      isBlowing: false,
      puffDetected: false,
      blowStartTime: 0,
      blowDuration: 0,
      minBlowDuration: 500, // kræv min. 100 ms høj energi
      puffCooldown: 500,
      lastPuffTime: 0,

      // Fysik
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.1,
      bounceDamping: 0.75,
      airResistance: 0.97,
      ballRadius: 15,
      ballEjected: false,

      // Parametre
      minHighHz: 2000,
      maxHighHz: 8000,
      relativeEnergyThreshold: 10,
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
      let sum = 0;
      for (let i = startBin; i < endBin; i++) sum += dataArray[i];
      return sum / (endBin - startBin);
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
      const relative = energy - this.baselineEnergy;

      // Justér baseline lidt hele tiden
      this.baselineEnergy =
        this.smoothingFactor * this.baselineEnergy + (1 - this.smoothingFactor) * energy;

      if (relative > this.relativeEnergyThreshold) {
        if (!this.isBlowing) {
          this.isBlowing = true;
          this.blowStartTime = now;
        }
        this.blowDuration = now - this.blowStartTime;
      } else {
        this.isBlowing = false;
        this.blowDuration = 0;
      }

      // Pust detekteres kun, hvis energien varer ved
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
        this.highFrequencyEnergy = 0.7 * this.highFrequencyEnergy + 0.3 * newEnergy;

        // Se om der er pust
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
  overflow: visible;
}

.playable-zone {
  position: absolute;
  left: -50px; right: -50px; top: -50px; bottom: 10px;
  background-color: rgba(33, 150, 243, 0.1);
  border: 2px dashed rgba(33, 150, 243, 0.4);
  pointer-events: none;
}

.safe-zone {
  position: absolute;
  left: 0; right: 0; top: 30px; bottom: 10px;
  background-color: rgba(76, 175, 80, 0.1);
  border-top: 2px dashed rgba(76, 175, 80, 0.4);
  border-bottom: 2px dashed rgba(76, 175, 80, 0.4);
  pointer-events: none;
}

.ball {
  width: 30px;
  height: 30px;
  background-color: red;
  border-radius: 50%;
  position: absolute;
  left: 35%;
  bottom: 10px;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}

/* Debugpanel styling */
.debug-panel {
  margin-top: 15px;
  padding: 10px;
  border: 1px solid #ccc;
  background: #f9f9f9;
  border-radius: 8px;
  width: 260px;
  text-align: left;
  font-family: monospace;
  font-size: 14px;
}

.error-message {
  margin-top: 20px;
  padding: 10px;
  color: #D8000C;
  background-color: #FFD2D2;
  border: 1px solid #D8000C;
  border-radius: 5px;
  max-width: 300px;
}
</style>
