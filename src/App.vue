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
      <button @click="startGame" v-if="!gameStarted">Start Spil</button>
      <div v-if="gameStarted">
        <p>Blæs i mikrofonen for at holde bolden i luften!</p>
        <!-- Viser nu den korrekte decibel-værdi -->
        <p>Lydniveau: {{ decibels.toFixed(1) }} dB</p>
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
      decibels: -100,
      
      // Physics variabler - 2D bevægelse
      ballX: 50, // Horisontal position (i procent af glassets bredde)
      ballY: 10, // Vertikal position (i pixels fra bunden)
      velocityX: 0, // Horisontal hastighed
      velocityY: 0, // Vertikal hastighed
      gravity: 0.1, // Nedsat fra 1.2 til 0.7 for langsommere fald
      bounceDamping: 0.75, // Dæmper hoppet ved nedslag
      wallBounceDamping: 0.75, // Dæmper ved vægstød (lavere = mindre bounce)
      airResistance: 0.97, // Luftmodstand (tættere på 1.0 = mindre modstand)
      dbThreshold: 82,
      
      // Animations variabler
      animationId: null,
      ballSetterX: null,
      ballSetterY: null,
      ballRadius: 15, // Radius i pixels
      ballEjected: false, // Marker om bolden er udstødt fra glasset
    };
  },
  mounted() {
    // Start bolden i bunden, centreret
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
        alert("Kunne ikke få adgang til mikrofonen. Tillad venligst adgang i din browser.");
      }
    },
    
    calculateDb(analyserNode) {
      const bufferLength = analyserNode.fftSize;
      const dataArray = new Float32Array(bufferLength);
      analyserNode.getFloatTimeDomainData(dataArray);

      let sumOfSquares = 0;
      for (let i = 0; i < bufferLength; i++) {
        sumOfSquares += dataArray[i] * dataArray[i];
      }
      
      const rms = Math.sqrt(sumOfSquares / bufferLength);
      
      if (rms === 0) return -100;

      const db = 20 * Math.log10(rms);
      return db + 100;
    },

    setBallPosition(x, y) {
      this.ballX = x;
      this.ballY = y;
      if (this.$refs.ballElement) {
        if (!this.ballSetterX) {
          // Fjern transform så vi kan bruge left direkte
          this.$refs.ballElement.style.transform = 'none';
          this.ballSetterX = gsap.quickSetter(this.$refs.ballElement, 'left', '%');
          this.ballSetterY = gsap.quickSetter(this.$refs.ballElement, 'bottom', 'px');
        }
        // Konverter x position (x er allerede i procent, men vi skal justere for boldens bredde)
        // For at centrere bolden: left = x - (ballRadius/glassWidth * 100)
        const glassWidth = 100;
        const leftPercent = x - (this.ballRadius / glassWidth * 100);
        this.ballSetterX(leftPercent);
        this.ballSetterY(y);
      }
    },

    spawnNewBall() {
      // Reset boldens position og hastighed
      this.ballX = 50;
      this.ballY = 10;
      this.velocityX = 0;
      this.velocityY = 0;
      this.ballEjected = false;
      this.setBallPosition(this.ballX, this.ballY);
    },

    gameLoop() {
      const glassWidth = 100; // Glassets bredde i pixels
      const glassHeight = 200; // Glassets højde i pixels
      const minY = 10; // Minimum Y position (bunden af glasset)
      const glassTop = glassHeight; // Toppen af glasset (200px)
      const maxHeight = 250; // Maksimal højde bolden kan nå
      const minX = (this.ballRadius / glassWidth) * 100; // Minimum X (venstre væg)
      const maxX = 100 - minX; // Maximum X (højre væg)
      const viewportBottom = -150; // Når bolden falder under dette, spawner der en ny
      
      const update = () => {
        // Opdater decibel-værdien
        this.decibels = this.calculateDb(this.analyser);

        // Tjek om bolden er ude af glasset horisontalt (tidligt så vi kan bruge det senere)
        const isOutsideGlass = this.ballX < minX || this.ballX > maxX;
        
        // Anvend kraft hvis blæser (kun vertikal, kun hvis bolden ikke er over max højde, og kun hvis den er i glasset og ikke er udstødt)
        if (this.decibels > this.dbThreshold && this.ballY < maxHeight && !isOutsideGlass && !this.ballEjected) {
          const forceStrength = (this.decibels - this.dbThreshold) / 20; // Øget divisor for mindre kraft (15 -> 20)
          this.velocityY += forceStrength * 0.3; // Reduceret kraftmultiplikator (1.5 -> 0.7)
          
          // Tilføj lidt tilfældig horisontal bevægelse når man blæser (som en bold i glas)
          // Kun hvis bolden er inde i glasset
          if (Math.abs(this.velocityY) > 2 && this.ballX >= minX && this.ballX <= maxX) {
            this.velocityX += (Math.random() - 0.5) * 0.3;
          }
        }
        // Hvis bolden er ude af glasset, kan den ikke få kraft fra pust

        // Anvend luftmodstand (bremser både horisontal og vertikal bevægelse)
        this.velocityX *= this.airResistance;
        this.velocityY *= this.airResistance;
        
        // Hvis bolden er på eller meget tæt på bunden, brems horisontal bevægelse ekstra
        if (this.ballY <= minY + 5) {
          this.velocityX *= 0.9; // Ekstra bremsning på bunden
        }

        // Anvend gravity (kun vertikal)
        this.velocityY -= this.gravity;

        // Opdater position
        this.ballX += this.velocityX;
        this.ballY += this.velocityY;
        
        // Begræns maksimal højde
        if (this.ballY > maxHeight) {
          this.ballY = maxHeight;
          if (this.velocityY > 0) {
            this.velocityY = 0;
          }
        }

        // Tjek om bolden rammer toppen af glasset - hvis ja, udstød den
        if (!this.ballEjected && this.ballY >= glassTop && this.ballY < glassTop + 10) {
          // Bolden har ramt toppen - udstød den
          this.ballEjected = true;
          // Giv bolden en kraftig push væk fra glasset (tilfældig retning men kraftig)
          const pushDirection = this.ballX < 50 ? -1 : 1; // Pusher væk fra center
          // Giv bolden en kraftig horisontal hastighed væk fra glasset
          this.velocityX = pushDirection * (8 + Math.random() * 2); // Kraftig hastighed væk (5-8 i retning væk fra center)
          // Sæt bolden lige over glasset
          this.ballY = glassTop + 5;
        }

        // Håndter collision med venstre og højre væg (kun hvis bolden er inde i glasset vertikalt og ikke er udstødt)
        if (!isOutsideGlass && !this.ballEjected && this.ballY >= minY && this.ballY <= glassTop + 50) {
          if (this.ballX <= minX) {
            this.ballX = minX;
            this.velocityX = -this.velocityX * this.wallBounceDamping;
          } else if (this.ballX >= maxX) {
            this.ballX = maxX;
            this.velocityX = -this.velocityX * this.wallBounceDamping;
          }
        }
        // Hvis bolden er ude af glasset horisontalt, kan den ikke ramme væggene igen

        // Håndter collision med bunden (kun hvis bolden er inde i glasset både horisontalt og vertikalt, og ikke er udstødt)
        if (!isOutsideGlass && !this.ballEjected && this.ballY <= minY && this.ballY >= viewportBottom && 
            this.ballX >= minX && this.ballX <= maxX) {
          this.ballY = minY;
          if (this.velocityY < 0) {
            this.velocityY = -this.velocityY * this.bounceDamping;
            
            // Kun tilføj horisontal bevægelse hvis bolden faktisk hopper (ikke bare ligger stille)
            if (Math.abs(this.velocityY) > 3) {
              // Ved kraftigt nedslag, tilføj lidt tilfældig horisontal bevægelse
              this.velocityX += (Math.random() - 0.8) * 0.8;
            }
            
            // Stop lille hopper og brems horisontal bevægelse på bunden
            if (Math.abs(this.velocityY) < 1.0) {
              this.velocityY = 0;
              // Brems horisontal bevægelse hurtigere når bolden er på bunden
              this.velocityX *= 0.8;
              // Stop helt hvis hastigheden bliver meget lille
              if (Math.abs(this.velocityX) < 0.3) {
                this.velocityX = 0;
              }
            }
          }
        }
        // Hvis bolden er ude af glasset, kan den ikke ramme bunden af glasset

        // Hvis bolden er udstødt, sikr at den ikke kan falde ned i glasset igen
        if (this.ballEjected && this.ballY < glassTop && this.ballY > minY) {
          // Hvis bolden falder ned gennem glasset, giv den ekstra horisontal hastighed væk
          if (this.ballX >= minX && this.ballX <= maxX) {
            // Bolden er direkte over glasset - skub den væk hurtigt
            const pushAway = this.ballX < 50 ? -3 : 3;
            this.velocityX = pushAway;
            // Flyt bolden væk fra glasset
            this.ballX = this.ballX < 50 ? minX - 5 : maxX + 5;
          }
        }

        // Tjek om bolden er faldet ud af viewporten (for langt ned)
        if (this.ballY < viewportBottom) {
          // Spawn ny bold
          this.spawnNewBall();
        }

        // Opdater bolden med GSAP
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

/* Blå zone der markerer hele området bolden kan befinde sig i */
.playable-zone {
  position: absolute;
  left: -50px; /* Udvidet lidt til siderne så den også dækker når bolden er ude */
  right: -50px;
  top: -50px; /* Fra toppen af glasset (200px fra bunden) op til maxHeight (250px) = 50px opad */
  bottom: 10px; /* Fra bunden af glasset (minY) */
  background-color: rgba(33, 150, 243, 0.15);
  border: 2px dashed rgba(33, 150, 243, 0.4);
  pointer-events: none;
  z-index: 0;
}

/* Grøn zone der markerer området hvor bolden kan bevæge sig */
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
  background-image: url('./pic/pngimg.com - ping_pong_PNG10364.png');
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  border-radius: 50%;
  position: absolute;
  left: 35%; /* Starter centreret (50% - 15% for radius) */
  bottom: 10px;
  transition: none; /* Fjern CSS transition så GSAP kan styre det */
  will-change: left, bottom; /* Optimering for animation */
  z-index: 2; /* Sørg for at bolden er over zonerne */
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>