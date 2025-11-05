<template>
  <div id="app" class="container">
    <h1>Pust bolden ud af glasset</h1>
    <p v-if="!micStarted">Giv tilladelse til mikrofonen for at starte.</p>

    <div class="glass-container">
      <div class="glass">
        <div class="ball" :style="{ transform: `translateY(${-ballPosition}px)` }"></div>
      </div>
    </div>

    <button @click="start" v-if="!micStarted">Start Mikrofon</button>

    <div v-if="micStarted" class="feedback">
      <div class="bar-container">
        <div class="bar" :style="{ width: blowPower * 100 + '%' }"></div>
      </div>
      <p>Pustestyrke: {{ Math.round(blowPower * 100) }}%</p>
    </div>

    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // Boldens fysik
      ballPosition: 0,
      ballVelocity: 0,

      // Lydanalyse
      blowPower: 0, // Den endelige, udjævnede pustestyrke (0-1)
      sustainedBlowCount: 0, // Tæller for vedvarende pust
      
      // App-tilstand
      micStarted: false,
      error: null,

      // Web Audio API objekter
      audioContext: null,
      analyser: null,
      animationFrameId: null,
    };
  },
  methods: {
    async start() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.micStarted = true;
        
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.analyser = this.audioContext.createAnalyser();
        
        // --- Vigtige indstillinger for præcis detektion ---
        this.analyser.fftSize = 1024; // Højere opløsning i frekvenser
        this.analyser.smoothingTimeConstant = 0.7; // Udjævner signalet

        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.analyser);

        this.gameLoop();
      } catch (err) {
        this.error = "Kunne ikke få adgang til mikrofonen. Tjek dine browser-tilladelser.";
        console.error(err);
      }
    },

    analyzeSound(dataArray) {
      const bufferLength = dataArray.length;

      // --- DEFINER FREKVENSOMRÅDER ---
      // Med fftSize=1024 og sampleRate=48000Hz er hver bin ca. 47Hz bred.
      const BASS_BINS = 5;       // 0 - 235Hz (Her ligger "poppet" fra pustet)
      const SPEECH_BINS_START = 6;  // 235Hz
      const SPEECH_BINS_END = 60;   // ~2800Hz (Her ligger energien fra tale)
      const FULL_SPECTRUM_AVG_BINS = 80; // Tjekker den generelle lydstyrke

      // Mål energien i de definerede områder
      let bassEnergy = 0;
      for (let i = 0; i < BASS_BINS; i++) {
        bassEnergy += dataArray[i];
      }
      bassEnergy /= BASS_BINS;

      let speechEnergy = 0;
      for (let i = SPEECH_BINS_START; i < SPEECH_BINS_END; i++) {
        speechEnergy += dataArray[i];
      }
      speechEnergy /= (SPEECH_BINS_END - SPEECH_BINS_START);
      
      let overallVolume = 0;
      for (let i = 0; i < FULL_SPECTRUM_AVG_BINS; i++) {
        overallVolume += dataArray[i];
      }
      overallVolume /= FULL_SPECTRUM_AVG_BINS;

      // --- KERNE LOGIK FOR PUST-DETEKTION ---
      // 1. Der skal være en vis minimumslydstyrke.
      // 2. Energien i bassen skal være markant højere end i tale-området.
      // 3. Bass-energien skal overstige en bestemt tærskel.
      const isBlowingCandidate = 
        overallVolume > 15 && 
        bassEnergy > speechEnergy * 1.5 && 
        bassEnergy > 50;

      // Gør detektoren mindre følsom overfor korte lyde ved at kræve vedvarende pust
      if (isBlowingCandidate) {
        this.sustainedBlowCount = Math.min(this.sustainedBlowCount + 1, 20); // Går hurtigt op
      } else {
        this.sustainedBlowCount = Math.max(this.sustainedBlowCount - 1, 0); // Falder langsomt
      }
      
      // Kun hvis pustet er vedvarende (counter > 5), beregner vi en reel styrke
      if (this.sustainedBlowCount > 5) {
        // Normaliser pustestyrken baseret på bass-energien.
        // Værdier er fundet ved eksperimentering.
        this.blowPower = Math.min((bassEnergy - 50) / 150, 1.0);
      } else {
        this.blowPower = 0;
      }
    },

    gameLoop() {
      const dataArray = new Uint8Array(this.analyser.frequencyBinCount);
      this.analyser.getByteFrequencyData(dataArray);
      
      this.analyzeSound(dataArray);

      // --- FYSIK-OPDATERING ---
      const gravity = 0.8;
      const blowForce = this.blowPower * 2.5;
      const drag = 0.98; // Luftmodstand

      // Tilføj kraft fra pustet
      this.ballVelocity += blowForce;
      // Træk tyngdekraft fra
      this.ballVelocity -= gravity;
      // Anvend luftmodstand
      this.ballVelocity *= drag;

      this.ballPosition += this.ballVelocity;

      // Sørg for at bolden ikke falder gennem bunden
      if (this.ballPosition < 0) {
        this.ballPosition = 0;
        this.ballVelocity = 0;
      }
      
      this.animationFrameId = requestAnimationFrame(this.gameLoop);
    },
  },
  beforeUnmount() {
    if (this.animationFrameId) {
      cancelAnimationFrame(this.animationFrameId);
    }
    if (this.audioContext) {
      this.audioContext.close();
    }
  },
};
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  font-family: sans-serif;
  background-color: #eef;
}
.glass-container {
  margin: 20px 0;
  perspective: 500px;
}
.glass {
  width: 150px;
  height: 300px;
  border: 5px solid #555;
  border-top: none;
  background: rgba(255, 255, 255, 0.6);
  border-radius: 0 0 15px 15px;
  position: relative;
  display: flex;
  align-items: flex-end;
  justify-content: center;
}
.ball {
  width: 60px;
  height: 60px;
  background-color: #d9534f;
  border-radius: 50%;
  position: absolute;
  bottom: 0;
  transition: transform 50ms linear;
}
.feedback {
  margin-top: 20px;
  width: 200px;
  text-align: center;
}
.bar-container {
  height: 20px;
  border: 1px solid #aaa;
  background-color: #ddd;
  border-radius: 10px;
  overflow: hidden;
}
.bar {
  height: 100%;
  background-color: #5cb85c;
  transition: width 0.1s linear;
  border-radius: 10px;
}
.error {
  color: #d9534f;
  margin-top: 15px;
}
button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  border: none;
  border-radius: 5px;
  background-color: #337ab7;
  color: white;
}
</style>```