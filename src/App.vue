<template>
  <div id="app">
    <div class="game-container">
      <div class="glass">
        <div class="ball" :style="{ bottom: ballPosition + 'px' }"></div>
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
export default {
  data() {
    return {
      gameStarted: false,
      audioContext: null,
      analyser: null,
      decibels: -100, // Startværdi for decibel (repræsenterer stilhed)
      ballPosition: 10,
      
      // "Fysik" variabler
      upwardForce: 4,
      gravity: 1.5,
      
      // Præcis tærskel i decibel, som ønsket
      dbThreshold: 60,
    };
  },
  methods: {
    async startGame() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.analyser = this.audioContext.createAnalyser();
        
        // Konfigurerer AnalyserNode for mere præcise dB-målinger
        this.analyser.fftSize = 2048;
        this.analyser.minDecibels = -90;
        this.analyser.maxDecibels = -10;
        this.analyser.smoothingTimeConstant = 0.85;

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
      
      if (rms === 0) return -100; // Undgå log(0) fejl, repræsenterer stilhed

      // Konverter RMS til dBFS (Decibels relative to full scale)
      // Værdien vil typisk være negativ. 0 dB er maks, lavere værdier er mere stille.
      // For at gøre det mere intuitivt (højere tal = højere lyd), lægger vi 100 til.
      const db = 20 * Math.log10(rms);
      return db + 100; // Justerer skalaen, så 0 er stilhed og ~100 er maks.
    },

    gameLoop() {
      const update = () => {
        // Opdater decibel-værdien i hver frame
        this.decibels = this.calculateDb(this.analyser);

        // --- Logik baseret på den præcise dB-tærskel ---
        if (this.decibels > this.dbThreshold) {
          this.ballPosition += this.upwardForce;
        } else {
          this.ballPosition -= this.gravity;
        }

        // Sørg for at bolden bliver inden for glassets grænser
        const minPosition = 10;
        const maxPosition = 170;
        this.ballPosition = Math.max(minPosition, Math.min(this.ballPosition, maxPosition));
        
        requestAnimationFrame(update);
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
  overflow: hidden;
}

.ball {
  width: 30px;
  height: 30px;
  background-color: darkorange;
  border-radius: 50%;
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>