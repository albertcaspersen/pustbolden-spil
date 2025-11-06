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
      // NYE: Opdeler energien for bedre analyse
      lowFrequencyEnergy: 0,
      highFrequencyEnergy: 0,
      combinedEnergy: 0,
      baseline: 0,
      prevEnergy: 0,
      baselineReady: false,

      // Pust-detektion
      isBlowing: false,
      lastPuffTime: 0,
      puffCooldown: 400, // Lidt kortere cooldown

      // Fysik
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.25,
      bounceDamping: 0.75,
      airResistance: 0.92,
      ballRadius: 15,

      // Parametre
      // FORBEDRET: Definerer frekvensbånd til analyse
      lowHzStart: 100,
      lowHzEnd: 1500,
      highHzStart: 3000,
      highHzEnd: 10000,

      // FORBEDRET: Tærskler for pust-detektion
      // Hvor meget energi over baseline kræves der?
      puffThreshold: 8,
      // Hvor stor skal den pludselige stigning være?
      onsetThreshold: 10,
      // Forholdet mellem høj- og lavfrekvent støj (et pust er ofte balanceret)
      balanceThreshold: 0.7,

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

        const stream = await navigator.mediaDevices.getUserMedia({
          audio: {
            // FORBEDRET: Moderne browsere understøtter disse til at fjerne ekko og konstant støj
            noiseSuppression: true,
            echoCancellation: true,
          },
        });
        const source = this.audioContext.createMediaStreamSource(stream);

        // JUSTERET: Gør filteret lidt bredere for at fange mere af pust-lyden
        const bandpass = this.audioContext.createBiquadFilter();
        bandpass.type = "bandpass";
        bandpass.frequency.value = 4000;
        bandpass.Q.value = 1; // Lavere Q-værdi = bredere filter

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.smoothingTimeConstant = 0.2;

        source.connect(bandpass);
        bandpass.connect(this.analyser);

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
          if (samples.length === 0) {
            this.baseline = 15;
          } else {
            const avg = samples.reduce((a, b) => a + b, 0) / samples.length;
            this.baseline = avg;
          }
          this.prevEnergy = this.baseline;
          console.log("✅ Kalibrering færdig. Baseline:", this.baseline.toFixed(1));
          resolve();
        }, 2500);
      });
    },

    // NY FUNKTION: Analyserer energien i flere frekvensbånd
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
      
      // Vægter den samlede energi. RMS (den generelle lydstyrke) tæller mest.
      const timeDomainArray = new Uint8Array(this.analyser.fftSize);
      this.analyser.getByteTimeDomainData(timeDomainArray);
      let sumSquares = 0.0;
      for (const amplitude of timeDomainArray) {
          const value = (amplitude / 128.0) - 1.0;
          sumSquares += value * value;
      }
      const rms = Math.sqrt(sumSquares / timeDomainArray.length) * 500;

      return {
        low: low,
        high: high,
        combined: (low + high) / 2 + rms,
      };
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

    // HELT NY LOGIK: Mere robust detektion af pust
    detectBlow(now, energies) {
      // 1. Juster langsomt baseline til den generelle baggrundsstøj
      this.baseline = this.baseline * 0.998 + energies.combined * 0.002;
      
      const energyAboveBaseline = energies.combined - this.baseline;
      const energyDelta = energies.combined - this.prevEnergy;
      this.prevEnergy = energies.combined;

      // Cooldown for at undgå flere pust lige efter hinanden
      if (now - this.lastPuffTime < this.puffCooldown) {
        return false;
      }
      
      // 2. Tjek for en pludselig, kraftig stigning i energien (et "onset")
      const isSuddenOnset = energyDelta > this.onsetThreshold;
      
      // 3. Tjek om den samlede energi er markant over baggrundsstøjen
      const isEnergyHighEnough = energyAboveBaseline > this.puffThreshold;

      // 4. Tjek om energien er "bredspektret" (både lav og høj frekvens)
      // Dette hjælper med at ignorere tale, som ofte har mere specifikke frekvenser.
      const highToLowRatio = energies.high / energies.low;
      const isBalanced = highToLowRatio > this.balanceThreshold && energies.low > 20;

      if (isSuddenOnset && isEnergyHighEnough && isBalanced) {
        this.lastPuffTime = now;
        console.log("💨 Pust detekteret!", {
            delta: energyDelta.toFixed(1),
            aboveBaseline: energyAboveBaseline.toFixed(1),
            ratio: highToLowRatio.toFixed(1),
        });
        return true;
      }
      
      return false;
    },

    gameLoop() {
      const maxHeight = 400;
      const minY = -200;
      const viewportBottom = -250;

      const update = () => {
        const now = performance.now();
        const energies = this.calculateEnergies();

        this.lowFrequencyEnergy = energies.low;
        this.highFrequencyEnergy = energies.high;
        this.combinedEnergy = energies.combined;

        const blowDetected = this.detectBlow(now, energies);

        if (blowDetected && this.ballY < maxHeight) {
          // Giver bolden et kraftigere og mere tilfredsstillende skub
          const force = Math.min((this.combinedEnergy - this.baseline) / 4, 15);
          this.velocityY += force;
          this.velocityX += (Math.random() - 0.5) * 0.4; // Lidt mere sideværts bevægelse
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
