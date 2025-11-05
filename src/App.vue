<template>
  <div id="app" class="container">
    <div class="main-content">
      <h1>Pust bolden ud af glasset</h1>

      <div class="glass-container">
        <div class="glass">
          <div class="ball" :style="{ transform: `translateY(${-ballPosition}px)` }"></div>
        </div>
      </div>

      <button @click="start" v-if="!micStarted" :disabled="isCalibrating">
        {{ isCalibrating ? 'Kalibrerer (vær helt stille)...' : 'Start Mikrofon' }}
      </button>

      <p v-if="error" class="error">{{ error }}</p>
    </div>

    <!-- VISUALIZER SEKTION -->
    <div v-if="micStarted && !isCalibrating" class="visualizer-section">
      <div class="power-feedback">
        <span class="power-label">Registreret Pustekraft</span>
        <div class="power-bar-container">
          <div class="power-bar" :style="{ width: blowPower * 100 + '%' }"></div>
        </div>
      </div>

      <div class="visualizer-container">
        <div 
          v-for="(value, index) in frequencyDataForViz" 
          :key="index" 
          class="freq-bar"
          :style="{ height: value + 'px' }"
          :class="{ 
            'bass-zone': isBassZone(index), 
            'speech-zone': isSpeechZone(index) 
          }"
        ></div>
      </div>
      <div class="legend">
        <div><span class="color-box bass-zone"></span> = Pust-zone (Bas)</div>
        <div><span class="color-box speech-zone"></span> = Tale/Støj-zone</div>
      </div>
    </div>

  </div>
</template>

<script>
const BASS_BINS_LOW = 1;
const BASS_BINS_HIGH = 5;
const SPEECH_BINS_LOW = 6;
const SPEECH_BINS_HIGH = 60;

export default {
  data() {
    return {
      ballPosition: 0,
      isCalibrating: false,
      baselineBass: 5, // Sæt en lav, sikker default
      blowPower: 0,
      micStarted: false,
      error: null,
      audioContext: null,
      analyser: null,
      animationFrameId: null,
      frequencyDataForViz: new Array(128).fill(0),
    };
  },
  methods: {
    isBassZone(index) { return index >= BASS_BINS_LOW && index <= BASS_BINS_HIGH; },
    isSpeechZone(index) { return index >= SPEECH_BINS_LOW && index <= SPEECH_BINS_HIGH; },
    
    async start() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: {
          // Anmod om at støj-undertrykkelse og lignende slås fra for at få det rå signal
          noiseSuppression: false,
          echoCancellation: false,
          autoGainControl: false,
        }});
        this.micStarted = true;
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.analyser = this.audioContext.createAnalyser();
        // --- FINJUSTEREDE VÆRDIER ---
        this.analyser.fftSize = 1024;
        this.analyser.smoothingTimeConstant = 0.2; // Lavere værdi for hurtigere reaktion
        this.analyser.minDecibels = -90; // Fang et bredere dynamisk område
        this.analyser.maxDecibels = -10;
        
        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.analyser);
        
        await this.calibrate();
        this.gameLoop();

      } catch (err) {
        this.error = "Mikrofonadgang blev nægtet. Genindlæs siden og prøv igen.";
      }
    },

    calibrate() {
      return new Promise(resolve => {
        this.isCalibrating = true;
        const calibrationSamples = [];
        const calibrationDuration = 2000;

        const interval = setInterval(() => {
          const dataArray = new Uint8Array(this.analyser.frequencyBinCount);
          this.analyser.getByteFrequencyData(dataArray);
          const bassEnergy = this.getFrequencyEnergy(dataArray, BASS_BINS_LOW, BASS_BINS_HIGH);
          calibrationSamples.push(bassEnergy);
        }, 100);

        setTimeout(() => {
          clearInterval(interval);
          if (calibrationSamples.length > 0) {
            const sum = calibrationSamples.reduce((a, b) => a + b, 0);
            this.baselineBass = Math.max(5, sum / calibrationSamples.length); // Sørg for at den ikke er 0
          }
          this.isCalibrating = false;
          resolve();
        }, calibrationDuration);
      });
    },

    getFrequencyEnergy(dataArray, lowBin, highBin) {
      let energy = 0;
      for (let i = lowBin; i <= highBin; i++) {
        energy += dataArray[i];
      }
      return energy / ((highBin - lowBin) + 1);
    },

    analyzeSound(dataArray) {
      if (this.isCalibrating) return 0;
      
      const currentBass = this.getFrequencyEnergy(dataArray, BASS_BINS_LOW, BASS_BINS_HIGH);
      const currentSpeech = this.getFrequencyEnergy(dataArray, SPEECH_BINS_LOW, SPEECH_BINS_HIGH);
      
      // --- NY, RATIO-BASERET LOGIK ---
      
      // 1. Støjfilter: Hvis der er for meget tale-støj, ignorer signalet.
      if (currentSpeech > currentBass * 0.8 && currentSpeech > 20) {
        return 0;
      }
      
      const bassRatio = currentBass / this.baselineBass;
      
      // 2. Aktiveringstærskel: Pustet skal være mindst 4x kraftigere end baggrundsstøjen.
      if (bassRatio < 4) {
        return 0;
      }
      
      // 3. Kraft-beregning: Map forholdet til en styrke mellem 0 og 1.
      // Et forhold på 4 er minimum (0% kraft), et forhold på 30 er max (100% kraft).
      const minRatio = 4;
      const maxRatio = 30;
      const power = (bassRatio - minRatio) / (maxRatio - minRatio);
      
      return Math.max(0, Math.min(power, 1.0)); // Sørg for at resultatet er mellem 0 og 1
    },

    gameLoop() {
      const dataArray = new Uint8Array(this.analyser.frequencyBinCount);
      this.analyser.getByteFrequencyData(dataArray);

      this.frequencyDataForViz = Array.from(dataArray.slice(0, 128));
      
      this.blowPower = this.analyzeSound(dataArray);

      // --- DIREKTE KOBLET FYSIK ---
      // Boldens position er nu en direkte afspejling af pustekraften.
      const glassHeight = 300;
      this.ballPosition = this.blowPower * glassHeight;
      
      this.animationFrameId = requestAnimationFrame(this.gameLoop);
    },
  },
  beforeUnmount() {
    if (this.animationFrameId) cancelAnimationFrame(this.animationFrameId);
    if (this.audioContext) this.audioContext.close();
  },
};
</script>

<style>
/* ... (Alle styles fra forrige svar kan genbruges præcist som de var) ... */
:root {
  --bass-color: #007bff;
  --speech-color: #ff9800;
  --power-color: #28a745;
  --default-bar-color: #ced4da;
}
body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
.container {
  display: flex; flex-direction: column; align-items: center; justify-content: flex-start;
  height: 100vh; background-color: #f4f7f6; text-align: center; box-sizing: border-box;
  padding: 20px 0; gap: 20px;
}
.main-content { display: flex; flex-direction: column; align-items: center; }
.glass-container { margin-bottom: 20px; }
.glass {
  width: 150px; height: 300px; border: 6px solid #4a4a4a; border-top: none;
  background: rgba(200, 200, 220, 0.3); border-radius: 0 0 20px 20px;
  position: relative; display: flex; align-items: flex-end; justify-content: center;
}
.ball {
  width: 60px; height: 60px; background-color: #e53935; border-radius: 50%;
  position: absolute; bottom: 0; 
  /* Gør overgangen øjeblikkelig for at matche den direkte kobling */
  transition: transform 0.05s linear;
}
button {
  padding: 12px 25px; font-size: 18px; cursor: pointer; border: none;
  border-radius: 8px; background-color: var(--bass-color); color: white; transition: all 0.2s;
}
button:disabled { background-color: #999; cursor: not-allowed; }
.error { color: #d9534f; margin-top: 15px; }
.visualizer-section { width: 100%; max-width: 600px; }
.power-feedback { margin-bottom: 15px; padding: 0 10px; }
.power-label { font-size: 14px; color: #333; font-weight: 500; display: block; margin-bottom: 5px; }
.power-bar-container {
  height: 25px; width: 100%; background-color: #e9ecef; border-radius: 5px;
  border: 1px solid #ccc; overflow: hidden;
}
.power-bar {
  height: 100%; background-color: var(--power-color);
  transition: width 0.05s linear; border-radius: 4px;
}
.visualizer-container {
  display: flex; align-items: flex-end; justify-content: center;
  height: 150px; background-color: #2c3e50; padding: 10px; border-radius: 8px; gap: 2px;
}
.freq-bar {
  flex-grow: 1; width: 2px; background-color: var(--default-bar-color);
  transition: height 0.05s linear;
}
.freq-bar.bass-zone { background-color: var(--bass-color); }
.freq-bar.speech-zone { background-color: var(--speech-color); }
.legend {
  display: flex; justify-content: center; gap: 20px; margin-top: 10px;
  font-size: 14px; color: #333;
}
.legend > div { display: flex; align-items: center; }
.color-box {
  width: 15px; height: 15px; border-radius: 3px; margin-right: 8px;
  border: 1px solid rgba(0,0,0,0.1);
}
.color-box.bass-zone { background-color: var(--bass-color); }
.color-box.speech-zone { background-color: var(--speech-color); }
</style>