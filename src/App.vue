<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <div class="playable-zone"></div>
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>
      
      <!-- Knappen vises kun før spillet starter -->
      <button @click="startGame" v-if="!gameStarted">Start Spil</button>
      
      <div v-if="gameStarted">
        <p>Blæs i mikrofonen for at holde bolden i luften!</p>
        <p>Højfrekvent Energi: {{ highFrequencyEnergy.toFixed(1) }}</p>
      </div>

      <!-- Viser en fejlbesked på skærmen, hvis noget går galt. Meget nyttigt til mobil debugging! -->
      <div v-if="errorMsg" class="error-message">
        <p><strong>Fejl:</strong> {{ errorMsg }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import { gsap } from 'gsap';

export default {
  data() {
    return {
      gameStarted: false,
      audioContext: null,
      analyser: null,
      highFrequencyEnergy: 0,
      errorMsg: null, // Til at vise fejl på skærmen
      
      // Fysik-variabler
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.1,
      bounceDamping: 0.75,
      wallBounceDamping: 0.75,
      airResistance: 0.97,
      energyThreshold: 15, // Sænket så den reagerer på realistisk energi
      minHighHz: 3000,
      maxHighHz: 8000,
      
      // Animations-variabler
      animationId: null,
      ballSetterX: null,
      ballSetterY: null,
      ballRadius: 15,
      ballEjected: false,
    };
  },
  mounted() {
    this.setBallPosition(50, 10);
  },
  beforeUnmount() {
    if (this.animationId) {
      cancelAnimationFrame(this.animationId);
    }
    if (this.audioContext && this.audioContext.state !== 'closed') {
      this.audioContext.close();
    }
  },
  methods: {
    // OPDATERET OG MERE ROBUST STARTGAME-METODE
    async startGame() {
      this.errorMsg = null; // Nulstil eventuelle gamle fejl

      try {
        // 1. Opret eller genoptag AudioContext ved bruger-klik (vigtigt for mobil!)
        if (!this.audioContext) {
          this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }

        // Hvis context er "suspenderet", skal den genoptages
        if (this.audioContext.state === 'suspended') {
          await this.audioContext.resume();
        }

        // 2. Anmod om mikrofonadgang (kræver HTTPS)
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

        // 3. Sæt lydanalysen op
        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;
        this.analyser.minDecibels = -90;
        this.analyser.maxDecibels = -10;
        this.analyser.smoothingTimeConstant = 0.15;

        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.analyser);
        
        // 4. Start spillet
        this.gameStarted = true;
        this.gameLoop();

      } catch (error) {
        console.error("Fejl ved start af spil:", error);
        // Gør fejlen synlig på skærmen
        if (error.name === 'NotAllowedError' || error.name === 'PermissionDeniedError') {
          this.errorMsg = "Du skal give adgang til mikrofonen for at spille.";
        } else if (error.name === 'NotFoundError') {
          this.errorMsg = "Ingen mikrofon fundet på din enhed.";
        } else if (window.location.protocol !== 'https:' && window.location.hostname !== 'localhost') {
           this.errorMsg = "Mikrofonadgang kræver en sikker forbindelse (HTTPS).";
        } else {
          this.errorMsg = "Kunne ikke starte spillet. Tjek dine browser-indstillinger og tilladelser.";
        }
      }
    },
    
    calculateHighFrequencyEnergy(analyserNode) {
      const binCount = analyserNode.frequencyBinCount; // fftSize/2
      const dataArray = new Uint8Array(binCount);
      analyserNode.getByteFrequencyData(dataArray);

      // Kortlæg ønsket Hz-interval til bin-indeks
      const nyquist = (this.audioContext?.sampleRate || 44100) / 2;
      const binSizeHz = nyquist / binCount; // Hz per bin
      const startBin = Math.max(0, Math.floor(this.minHighHz / binSizeHz));
      const endBin = Math.min(binCount, Math.ceil(this.maxHighHz / binSizeHz));

      const numberOfBins = Math.max(1, endBin - startBin);
      let sumOfEnergy = 0;
      for (let i = startBin; i < endBin; i++) {
        sumOfEnergy += dataArray[i]; // 0..255
      }
      const averageEnergy = sumOfEnergy / numberOfBins;
      return isNaN(averageEnergy) ? 0 : averageEnergy;
    },

    setBallPosition(x, y) {
      if (!this.$refs.ballElement) return;
      if (!this.ballSetterX) {
        this.$refs.ballElement.style.transform = 'none';
        this.ballSetterX = gsap.quickSetter(this.$refs.ballElement, 'left', '%');
        this.ballSetterY = gsap.quickSetter(this.$refs.ballElement, 'bottom', 'px');
      }
      const glassWidth = 100;
      const leftPercent = x - (this.ballRadius / glassWidth * 100);
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
      // ... (Resten af din gameLoop-kode er fin og behøver ingen ændringer)
      // ... (Kopieret fra din oprindelige kode for fuldstændighedens skyld)
      const glassWidth = 100;
      const glassHeight = 200;
      const minY = 10;
      const glassTop = glassHeight;
      const maxHeight = 250;
      const minX = (this.ballRadius / glassWidth) * 100;
      const maxX = 100 - minX;
      const viewportBottom = -150;
      
      const update = () => {
        this.highFrequencyEnergy = this.calculateHighFrequencyEnergy(this.analyser);
        const isOutsideGlass = this.ballX < minX || this.ballX > maxX;
        
        if (this.highFrequencyEnergy > this.energyThreshold && this.ballY < maxHeight && !isOutsideGlass && !this.ballEjected) {
          const forceStrength = (this.highFrequencyEnergy - this.energyThreshold) / 50;
          this.velocityY += forceStrength * 0.7;
          
          if (Math.abs(this.velocityY) > 2 && this.ballX >= minX && this.ballX <= maxX) {
            this.velocityX += (Math.random() - 0.5) * 0.3;
          }
        }

        this.velocityX *= this.airResistance;
        this.velocityY *= this.airResistance;
        if (this.ballY <= minY + 5) this.velocityX *= 0.9;
        this.velocityY -= this.gravity;
        this.ballX += this.velocityX;
        this.ballY += this.velocityY;
        
        if (this.ballY > maxHeight) {
          this.ballY = maxHeight;
          if (this.velocityY > 0) this.velocityY = 0;
        }

        if (!this.ballEjected && this.ballY >= glassTop && this.ballY < glassTop + 10) {
          this.ballEjected = true;
          const pushDirection = this.ballX < 50 ? -1 : 1;
          this.velocityX = pushDirection * (8 + Math.random() * 2);
          this.ballY = glassTop + 5;
        }

        if (!isOutsideGlass && !this.ballEjected && this.ballY >= minY && this.ballY <= glassTop + 50) {
          if (this.ballX <= minX) {
            this.ballX = minX;
            this.velocityX = -this.velocityX * this.wallBounceDamping;
          } else if (this.ballX >= maxX) {
            this.ballX = maxX;
            this.velocityX = -this.velocityX * this.wallBounceDamping;
          }
        }

        if (!isOutsideGlass && !this.ballEjected && this.ballY <= minY && this.ballY >= viewportBottom && this.ballX >= minX && this.ballX <= maxX) {
          this.ballY = minY;
          if (this.velocityY < 0) {
            this.velocityY = -this.velocityY * this.bounceDamping;
            if (Math.abs(this.velocityY) > 3) this.velocityX += (Math.random() - 0.8) * 0.8;
            if (Math.abs(this.velocityY) < 1.0) {
              this.velocityY = 0;
              this.velocityX *= 0.8;
              if (Math.abs(this.velocityX) < 0.3) this.velocityX = 0;
            }
          }
        }

        if (this.ballEjected && this.ballY < glassTop && this.ballY > minY) {
          if (this.ballX >= minX && this.ballX <= maxX) {
            const pushAway = this.ballX < 50 ? -3 : 3;
            this.velocityX = pushAway;
            this.ballX = this.ballX < 50 ? minX - 5 : maxX + 5;
          }
        }

        if (this.ballY < viewportBottom) {
          this.spawnNewBall();
        }

        this.setBallPosition(this.ballX, this.ballY);
        this.animationId = requestAnimationFrame(update);
      };
      update();
    },
  },
};
</script>

<style>
/* ... din CSS er fin ... */
/* Tilføj stil for fejlbeskeden */
.error-message {
  margin-top: 20px;
  padding: 10px;
  color: #D8000C; /* Mørkerød tekst */
  background-color: #FFD2D2; /* Lys rød baggrund */
  border: 1px solid #D8000C;
  border-radius: 5px;
  max-width: 300px;
}

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
  background-color: rgba(33, 150, 243, 0.15);
  border: 2px dashed rgba(33, 150, 243, 0.4);
  pointer-events: none;
  z-index: 0;
}

.safe-zone {
  position: absolute;
  left: 0; right: 0; top: 30px; bottom: 10px;
  background-color: rgba(76, 175, 80, 0.2);
  border-top: 2px dashed rgba(76, 175, 80, 0.5);
  border-bottom: 2px dashed rgba(76, 175, 80, 0.5);
  pointer-events: none;
  z-index: 1;
}

.ball {
  width: 30px; height: 30px;
  background-color: red;
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  border-radius: 50%;
  position: absolute;
  left: 35%; bottom: 10px;
  transition: none;
  will-change: left, bottom;
  z-index: 2;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>