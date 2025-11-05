<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <div class="playable-zone"></div>
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>

      <!-- Start-knap -->
      <button @click="startGame" v-if="!gameStarted && !calibrating">Start Spil</button>

      <!-- Kalibreringsbesked -->
      <div v-if="calibrating">
        <p>🤫 Kalibrerer... vær stille i 2 sekunder</p>
      </div>

      <!-- Spil og debugpanel -->
      <div v-if="gameStarted">
        <p>🎯 Blæs i mikrofonen for at holde bolden i luften!</p>

        <!-- Debug info -->
        <div class="debug-panel">
          <p><strong>Debug Info</strong></p>
          <p>Lydenergi: {{ highFrequencyEnergy.toFixed(1) }}</p>
          <p>Baseline: {{ baselineEnergy.toFixed(1) }}</p>
          <p>Relative Energi: {{ (highFrequencyEnergy - baselineEnergy).toFixed(1) }}</p>
          <p>DeltaE: {{ (highFrequencyEnergy - prevEnergy).toFixed(1) }}</p>
          <p>Threshold (Rel/Δ): {{ relativeEnergyThreshold }} / {{ deltaThreshold }}</p>
          <p>Pust-Status: <strong>{{ isBlowing ? "💨 Registreret" : "..." }}</strong></p>
        </div>
      </div>

      <!-- Fejlbesked -->
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
      // Spiltilstand
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
      lastPuffTime: 0,
      isBlowing: false,

      // Boldfysik
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.06,
      bounceDamping: 0.75,
      wallBounceDamping: 0.75,
      airResistance: 0.985,
      ballRadius: 15,
      ballEjected: false,

      // Parametre
      minHighHz: 1000,
      maxHighHz: 8000,
      relativeEnergyThreshold: 6,
      deltaThreshold: 3,
      puffCooldown: 150,
      smoothingFactor: 0.995, // baseline justering
      noiseFloorClamp: 0.05,

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
          this.audioContext = new (window.AudioContext ||
            window.webkitAudioContext)();
        }

        if (this.audioContext.state === "suspended") {
          await this.audioContext.resume();
        }

        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const source = this.audioContext.createMediaStreamSource(stream);

        const bandpass = this.audioContext.createBiquadFilter();
        bandpass.type = "bandpass";
        bandpass.frequency.value = 3000;
        bandpass.Q.value = 1.5;

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.smoothingTimeConstant = 0.15;

        source.connect(bandpass);
        bandpass.connect(this.analyser);

        // Kalibrer først
        this.calibrating = true;
        await this.measureBaseline();
        this.calibrating = false;
        this.baselineReady = true;

        this.gameStarted = true;
        this.gameLoop();
      } catch (error) {
        console.error("Fejl ved start:", error);
        if (
          error.name === "NotAllowedError" ||
          error.name === "PermissionDeniedError"
        ) {
          this.errorMsg = "Du skal give adgang til mikrofonen for at spille.";
        } else if (error.name === "NotFoundError") {
          this.errorMsg = "Ingen mikrofon fundet på din enhed.";
        } else if (
          window.location.protocol !== "https:" &&
          window.location.hostname !== "localhost"
        ) {
          this.errorMsg = "Mikrofonadgang kræver en sikker (HTTPS) forbindelse.";
        } else {
          this.errorMsg =
            "Kunne ikke starte spillet. Tjek browserens tilladelser.";
        }
      }
    },

    async measureBaseline() {
      return new Promise((resolve) => {
        const samples = [];
        const measure = () => {
          const energy = this.calculateHighFrequencyEnergy(this.analyser);
          samples.push(energy);
        };
        const interval = setInterval(measure, 100);
        setTimeout(() => {
          clearInterval(interval);
          const avg = samples.reduce((a, b) => a + b, 0) / samples.length;
          this.baselineEnergy = avg;
          console.log("Baseline sat til:", avg.toFixed(1));
          resolve(avg);
        }, 2000);
      });
    },

    calculateHighFrequencyEnergy(analyserNode) {
      const binCount = analyserNode.frequencyBinCount;
      const dataArray = new Uint8Array(binCount);
      analyserNode.getByteFrequencyData(dataArray);

      const nyquist = (this.audioContext?.sampleRate || 44100) / 2;
      const binSizeHz = nyquist / binCount;
      const startBin = Math.max(0, Math.floor(this.minHighHz / binSizeHz));
      const endBin = Math.min(binCount, Math.ceil(this.maxHighHz / binSizeHz));

      let sum = 0;
      for (let i = startBin; i < endBin; i++) sum += dataArray[i];
      return sum / (endBin - startBin);
    },

    setBallPosition(x, y) {
      if (!this.$refs.ballElement) return;
      if (!this.ballSetterX) {
        this.$refs.ballElement.style.transform = "none";
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

    gameLoop() {
      const glassWidth = 100;
      const glassHeight = 200;
      const minY = 10;
      const glassTop = glassHeight;
      const maxHeight = 250;
      const minX = (this.ballRadius / glassWidth) * 100;
      const maxX = 100 - minX;
      const viewportBottom = -150;

      const update = () => {
        const now = performance.now();
        const newEnergy = this.calculateHighFrequencyEnergy(this.analyser);
        const smoothed = 0.7 * this.highFrequencyEnergy + 0.3 * newEnergy;
        const deltaE = smoothed - this.prevEnergy;
        this.prevEnergy = smoothed;
        this.highFrequencyEnergy = smoothed;

        // Dynamisk baseline
        if (this.baselineReady) {
          const relativeEnergy = smoothed - this.baselineEnergy;
          if (deltaE < 2 && relativeEnergy < this.relativeEnergyThreshold) {
            const diff = smoothed - this.baselineEnergy;
            this.baselineEnergy += diff * (1 - this.smoothingFactor);
            this.baselineEnergy = Math.max(
              this.baselineEnergy - this.noiseFloorClamp,
              Math.min(this.baselineEnergy + this.noiseFloorClamp, smoothed)
            );
          }

          const isOutsideGlass = this.ballX < minX || this.ballX > maxX;

          // 💨 Detektion af pust
          if (
            relativeEnergy > this.relativeEnergyThreshold &&
            deltaE > this.deltaThreshold &&
            now - this.lastPuffTime > this.puffCooldown &&
            this.ballY < maxHeight &&
            !isOutsideGlass &&
            !this.ballEjected
          ) {
            const forceStrength =
              (relativeEnergy - this.relativeEnergyThreshold) / 25;
            this.velocityY += forceStrength * 1.2;
            this.velocityX += (Math.random() - 0.5) * 0.3;
            this.lastPuffTime = now;
            this.isBlowing = true;
          } else {
            this.isBlowing = false;
          }
        }

        // Fysik
        this.velocityX *= this.airResistance;
        this.velocityY *= this.airResistance;
        this.velocityY -= this.gravity;
        this.ballX += this.velocityX;
        this.ballY += this.velocityY;

        if (this.ballY > maxHeight) {
          this.ballY = maxHeight;
          if (this.velocityY > 0) this.velocityY = 0;
        }

        if (!this.ballEjected && this.ballY >= glassTop && this.ballY < glassTop + 10) {
          this.ballEjected = true;
          const pushDir = this.ballX < 50 ? -1 : 1;
          this.velocityX = pushDir * (8 + Math.random() * 2);
          this.ballY = glassTop + 5;
        }

        if (!this.ballEjected && this.ballY <= minY) {
          this.ballY = minY;
          if (this.velocityY < 0) {
            this.velocityY = -this.velocityY * this.bounceDamping;
            if (Math.abs(this.velocityY) < 1.0) this.velocityY = 0;
          }
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
