<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <div class="playable-zone"></div>
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>

      <button @click="startGame" v-if="!gameStarted && !calibrating">Start Spil</button>

      <div v-if="calibrating"></div>
      <div v-if="gameStarted"></div>

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

      audioContext: null,
      analyser: null,
      filter: null, // 🧩 NYT: Band-pass filter
      lowFrequencyEnergy: 0,
      highFrequencyEnergy: 0,
      combinedEnergy: 0,
      baseline: 0,
      prevEnergy: 0,
      baselineReady: false,

      isBlowing: false,
      blowFrames: 0, // 🧩 NYT: Til vedvarende pust-detektion
      blowThresholdFrames: 1, // Ca. 250–300 ms
      lastPuffTime: 0,
      puffCooldown: 400,

      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.2,
      bounceDamping: 0.75,
      airResistance: 0.92,
      ballRadius: 15,

      lowHzStart: 300,
      lowHzEnd: 2000,
      highHzStart: 2000,
      highHzEnd: 12000,

      puffThreshold: 1,
      onsetThreshold: 3,
      balanceThreshold: 0.6,

      // 🧩 NYT: Glidende gennemsnit
      energyHistory: [], 
      smoothWindow: 5,

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

        const stream = await navigator.mediaDevices.getUserMedia({
          audio: {
            noiseSuppression: true,
            echoCancellation: true,
          },
        });
        const source = this.audioContext.createMediaStreamSource(stream);

        // 🧩 NYT: Tilføj band-pass filter for at isolere pustefrekvenser
        this.filter = this.audioContext.createBiquadFilter();
        this.filter.type = "bandpass";
        this.filter.frequency.value = 1500;
        this.filter.Q.value = 1.5;

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.smoothingTimeConstant = 0.3;

        source.connect(this.filter);
        this.filter.connect(this.analyser);

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
          const { combined } = this.calculateEnergies();
          if (!isNaN(combined) && combined > 0.1) {
            samples.push(combined);
          }
        };

        const interval = setInterval(measure, 100);
        setTimeout(() => {
          clearInterval(interval);
          this.baseline = samples.length
            ? samples.reduce((a, b) => a + b, 0) / samples.length
            : 15;
          this.prevEnergy = this.baseline;
          console.log("✅ Kalibrering færdig. Baseline:", this.baseline.toFixed(1));
          resolve();
        }, 2500);
      });
    },

    calculateEnergies() {
      const dataArray = new Uint8Array(this.analyser.frequencyBinCount);
      this.analyser.getByteFrequencyData(dataArray);
      const nyquist = this.audioContext.sampleRate / 2;
      const binSizeHz = nyquist / this.analyser.frequencyBinCount;

      const getAvgEnergy = (startHz, endHz) => {
        const startBin = Math.floor(startHz / binSizeHz);
        const endBin = Math.ceil(endHz / binSizeHz);
        let sum = 0;
        for (let i = startBin; i < endBin; i++) {
          sum += dataArray[i];
        }
        return sum / (endBin - startBin);
      };

      const low = getAvgEnergy(this.lowHzStart, this.lowHzEnd);
      const high = getAvgEnergy(this.highHzStart, this.highHzEnd);

      const timeDomainArray = new Uint8Array(this.analyser.fftSize);
      this.analyser.getByteTimeDomainData(timeDomainArray);
      let sumSquares = 0.0;
      for (const amplitude of timeDomainArray) {
        const value = (amplitude / 128.0) - 1.0;
        sumSquares += value * value;
      }
      const rms = Math.sqrt(sumSquares / timeDomainArray.length) * 500;

      let combined = (low + high) / 2 + rms;

      // 🧩 NYT: Glidende gennemsnit for at udjævne peaks
      this.energyHistory.push(combined);
      if (this.energyHistory.length > this.smoothWindow)
        this.energyHistory.shift();
      combined =
        this.energyHistory.reduce((a, b) => a + b, 0) /
        this.energyHistory.length;

      return { low, high, combined };
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

    detectBlow(now, energies) {
      // 🧩 NYT: adaptiv baseline med langsom justering
      this.baseline = this.baseline * 0.995 + energies.combined * 0.005;

      const energyAboveBaseline = energies.combined - this.baseline;
      const energyDelta = energies.combined - this.prevEnergy;
      this.prevEnergy = energies.combined;

      const highToLowRatio = energies.low > 0 ? energies.high / energies.low : 0;
      const isBalanced = highToLowRatio > this.balanceThreshold;
      const hasMinimumEnergy = energies.low > 25 && energies.high > 20;
      const isSuddenOnset = energyDelta > this.onsetThreshold;
      const isEnergyHighEnough = energyAboveBaseline > this.puffThreshold;

      const blowNow =
        isEnergyHighEnough && isBalanced && hasMinimumEnergy && isSuddenOnset;

      // 🧩 NYT: kræv vedvarende pust
      if (blowNow) {
        this.blowFrames++;
      } else {
        this.blowFrames = Math.max(this.blowFrames - 1, 0);
      }

      const sustained = this.blowFrames >= this.blowThresholdFrames;

      if (sustained && now - this.lastPuffTime > this.puffCooldown) {
        this.lastPuffTime = now;
        this.blowFrames = 0;
        console.log("💨 Pust detekteret! (stabil)");
        return true;
      }

      return false;
    },

    gameLoop() {
      const minY = -200;
      const viewportBottom = -250;

      const update = () => {
        const now = performance.now();
        const energies = this.calculateEnergies();

        this.lowFrequencyEnergy = energies.low;
        this.highFrequencyEnergy = energies.high;
        this.combinedEnergy = energies.combined;

        const blowDetected = this.detectBlow(now, energies);

        if (blowDetected) {
          const force = Math.min(
            (this.combinedEnergy - this.baseline) / 2.5,
            20
          );
          this.velocityY += force;
          this.velocityX += (Math.random() - 0.5) * 0.4;
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
  border: #D8000C 1px solid;
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
  /*background-color: rgba(0, 255, 0, 0.2);
  border: 2px solid rgba(0, 255, 0, 0.5);*/
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