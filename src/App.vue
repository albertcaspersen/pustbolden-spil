<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <!-- Blå div der markerer området bolden kan befinde sig i -->
        <div class="playable-zone"></div>
        <!-- Grøn div der markerer området -->
        <div class="safe-zone"></div>
        <div class="ball" ref="ballElement"></div>
      </div>
      <!-- KORREKTION: "vif" er rettet til "v-if" for at knappen virker -->
      <button @click="startGame" v-if="!gameStarted">Start Spil</button>
      <div v-if="gameStarted">
        <p>Blæs i mikrofonen for at holde bolden i luften!</p>
        <!-- Viser nu energien i de høje frekvenser -->
        <p>Højfrekvent Energi: {{ highFrequencyEnergy.toFixed(1) }}</p>
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
      highFrequencyEnergy: 0, // Måler nu energi i høje frekvenser i stedet for generel dB
      
      // Physics variabler
      ballX: 50,
      ballY: 10,
      velocityX: 0,
      velocityY: 0,
      gravity: 0.1,
      bounceDamping: 0.75,
      wallBounceDamping: 0.75,
      airResistance: 0.97,
      energyThreshold: 50, // Tærskelværdi for hvornår pust skal registreres. Juster denne efter behov.
      
      // Animations variabler
      animationId: null,
      ballSetterX: null,
      ballSetterY: null,
      ballRadius: 15,
      ballEjected: false,
    };
  },
  mounted() {
    this.ballX = 50;
    this.ballY = 10;
    this.ballEjected = false;
    this.setBallPosition(this.ballX, this.ballY);
  },
  beforeUnmount() {
    if (this.animationId) {
      cancelAnimationFrame(this.animationId);
    }
    if (this.audioContext) {
      this.audioContext.close();
    }
  },
  methods: {
    async startGame() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.analyser = this.audioContext.createAnalyser();
        
        this.analyser.fftSize = 2048;
        this.analyser.minDecibels = -90;
        this.analyser.maxDecibels = -10;
        this.analyser.smoothingTimeConstant = 0.15;

        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.analyser);
        this.gameStarted = true;
        this.gameLoop();
      } catch (error) {
        console.error("Fejl ved adgang til mikrofon:", error);
        alert("Kunne ikke få adgang til mikrofonen. Tillad venligst adgang i din browser og sørg for at siden kører på localhost eller https.");
      }
    },
    
    // Metode til at beregne energien i et specifikt (højt) frekvensområde
    calculateHighFrequencyEnergy(analyserNode) {
      const bufferLength = analyserNode.frequencyBinCount; // Er halvdelen af fftSize (1024)
      const dataArray = new Uint8Array(bufferLength);
      analyserNode.getByteFrequencyData(dataArray);

      // === HER STYRER DU FREKVENSOMRÅDET ===
      // Hver 'bin' dækker ca. 23.4 Hz (ved 48000 Hz sample rate).
      // Vi lytter fra bin 200 til 800 for at fange pustelyde.
      const startBin = 200; // ca. 4.7 kHz
      const endBin = 800;   // ca. 18.7 kHz
      // ======================================

      const numberOfBins = endBin - startBin;
      let sumOfEnergy = 0;
      for (let i = startBin; i < endBin; i++) {
        sumOfEnergy += dataArray[i];
      }
      
      const averageEnergy = sumOfEnergy / numberOfBins;
      
      return isNaN(averageEnergy) ? 0 : averageEnergy;
    },

    setBallPosition(x, y) {
      this.ballX = x;
      this.ballY = y;
      if (this.$refs.ballElement) {
        if (!this.ballSetterX) {
          this.$refs.ballElement.style.transform = 'none';
          this.ballSetterX = gsap.quickSetter(this.$refs.ballElement, 'left', '%');
          this.ballSetterY = gsap.quickSetter(this.$refs.ballElement, 'bottom', 'px');
        }
        const glassWidth = 100;
        const leftPercent = x - (this.ballRadius / glassWidth * 100);
        this.ballSetterX(leftPercent);
        this.ballSetterY(y);
      }
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
        // Opdater med højfrekvent energi
        this.highFrequencyEnergy = this.calculateHighFrequencyEnergy(this.analyser);

        const isOutsideGlass = this.ballX < minX || this.ballX > maxX;
        
        // Anvend kraft baseret på højfrekvent energi
        if (this.highFrequencyEnergy > this.energyThreshold && this.ballY < maxHeight && !isOutsideGlass && !this.ballEjected) {
          const forceStrength = (this.highFrequencyEnergy - this.energyThreshold) / 50;
          this.velocityY += forceStrength * 0.7;
          
          if (Math.abs(this.velocityY) > 2 && this.ballX >= minX && this.ballX <= maxX) {
            this.velocityX += (Math.random() - 0.5) * 0.3;
          }
        }

        this.velocityX *= this.airResistance;
        this.velocityY *= this.airResistance;
        
        if (this.ballY <= minY + 5) {
          this.velocityX *= 0.9;
        }

        this.velocityY -= this.gravity;

        this.ballX += this.velocityX;
        this.ballY += this.velocityY;
        
        if (this.ballY > maxHeight) {
          this.ballY = maxHeight;
          if (this.velocityY > 0) {
            this.velocityY = 0;
          }
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

        if (!isOutsideGlass && !this.ballEjected && this.ballY <= minY && this.ballY >= viewportBottom && 
            this.ballX >= minX && this.ballX <= maxX) {
          this.ballY = minY;
          if (this.velocityY < 0) {
            this.velocityY = -this.velocityY * this.bounceDamping;
            
            if (Math.abs(this.velocityY) > 3) {
              this.velocityX += (Math.random() - 0.8) * 0.8;
            }
            
            if (Math.abs(this.velocityY) < 1.0) {
              this.velocityY = 0;
              this.velocityX *= 0.8;
              if (Math.abs(this.velocityX) < 0.3) {
                this.velocityX = 0;
              }
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
  overflow: visible; /* Tillad bolden at komme ud over glasset */
}

.playable-zone {
  position: absolute;
  left: -50px;
  right: -50px;
  top: -50px;
  bottom: 10px;
  background-color: rgba(33, 150, 243, 0.15);
  border: 2px dashed rgba(33, 150, 243, 0.4);
  pointer-events: none;
  z-index: 0;
}

.safe-zone {
  position: absolute;
  left: 0;
  right: 0;
  top: 30px;
  bottom: 10px;
  background-color: rgba(76, 175, 80, 0.2);
  border-top: 2px dashed rgba(76, 175, 80, 0.5);
  border-bottom: 2px dashed rgba(76, 175, 80, 0.5);
  pointer-events: none;
  z-index: 1;
}

.ball {
  width: 30px;
  height: 30px;
  /* Husk at du skal have et billede i den angivne sti for at dette virker */
  /* background-image: url('./pic/pngimg.com - ping_pong_PNG10364.png'); */
  background-color: red; /* Fallback farve, hvis billedet ikke findes */
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  border-radius: 50%;
  position: absolute;
  left: 35%;
  bottom: 10px;
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