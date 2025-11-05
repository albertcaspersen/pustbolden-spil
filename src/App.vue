<template>
  <div id="app" class="container">
    <h1>Pust bolden ud af glasset</h1>
    <p v-if="!micStarted">Klik på knappen og giv tilladelse til at bruge mikrofonen for at starte.</p>
    
    <div class="glass-container">
      <div class="glass">
        <!-- Boldens position styres nu af en CSS variabel for en mere jævn animation -->
        <div class="ball" :style="{ '--ball-bottom': ballPosition + 'px' }"></div>
      </div>
    </div>
    
    <button @click="startMic" v-if="!micStarted">Start Spillet</button>
    <div v-if="micStarted" class="info">
      <p>Pust i mikrofonen!</p>
      <!-- Viser den beregnede pustestyrke, godt til debugging og feedback -->
      <p>Pustestyrke: {{ Math.round(blowPower * 100) }}%</p>
    </div>

    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      ballPosition: 0, // Position i pixels fra bunden
      ballVelocity: 0, // Hastighed for mere realistisk bevægelse
      blowPower: 0, // Hvor kraftigt der pustes (værdi mellem 0 og 1)
      micStarted: false,
      error: null,
      audioContext: null,
      analyser: null,
      animationFrameId: null,
    };
  },
  methods: {
    async startMic() {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        this.error = "Din browser understøtter desværre ikke mikrofon-adgang.";
        return;
      }
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.micStarted = true;
        this.error = null;

        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.analyser = this.audioContext.createAnalyser();
        
        // Opsætning af lydanalysen
        this.analyser.fftSize = 512; // Finere frekvensopløsning
        this.analyser.smoothingTimeConstant = 0.6; // Udjævner signalet en smule

        const source = this.audioContext.createMediaStreamSource(stream);
        source.connect(this.analyser);

        // Start spil-loopet
        this.gameLoop();

      } catch (err) {
        this.error = "Adgang til mikrofon blev nægtet. Tjek venligst dine browser-indstillinger. Fejl: " + err.message;
      }
    },

    detectBlow(frequencyData) {
      // DEFINER FREKVENSOMRÅDER (afhænger af fftSize)
      // Med fftSize = 512 har vi 256 frekvens-bins.
      // SampleRate (typisk 48000Hz) / fftSize = frekvens pr. bin (ca. 93.75 Hz)
      const lowPassBins = 4;      // Bassen/trykket fra pustet (0 - ca. 375 Hz)
      const midSpeechBins = 20;   // Mellemtone-området for tale (ca. 375 Hz - 1875 Hz)
      
      // Beregn gennemsnitsenergien i de definerede områder
      let lowFreqAvg = 0;
      for (let i = 0; i < lowPassBins; i++) {
        lowFreqAvg += frequencyData[i];
      }
      lowFreqAvg /= lowPassBins;

      let midFreqAvg = 0;
      for (let i = lowPassBins; i < midSpeechBins; i++) {
        midFreqAvg += frequencyData[i];
      }
      midFreqAvg /= (midSpeechBins - lowPassBins);

      // --- KERNE LOGIK ---
      // Et pust er kendetegnet ved at have en stærk bas (lowFreqAvg)
      // og samtidig være stærkere i bassen end i mellemtone-området for tale.
      const isBlowing = lowFreqAvg > 80 && lowFreqAvg > midFreqAvg * 1.5;

      if (isBlowing) {
        // Normaliser pustestyrken til en værdi mellem 0 og ca. 1
        this.blowPower = Math.min(lowFreqAvg / 150, 1);
      } else {
        this.blowPower = 0;
      }
    },

    gameLoop() {
      // Hent frekvensdata fra mikrofonen
      const bufferLength = this.analyser.frequencyBinCount;
      const frequencyData = new Uint8Array(bufferLength);
      this.analyser.getByteFrequencyData(frequencyData);

      // Kør detektionslogikken
      this.detectBlow(frequencyData);

      // --- FYSIK FOR BOLDEN ---
      const gravity = 0.5;
      const blowForce = this.blowPower * 1.5; // Juster denne for at ændre, hvor "kraftigt" man puster

      // Tilføj pustekraft til hastigheden
      if (this.blowPower > 0.1) {
        this.ballVelocity += blowForce;
      }

      // Anvend tyngdekraft
      this.ballVelocity -= gravity;
      
      // Anvend luftmodstand for at dæmpe bevægelsen
      this.ballVelocity *= 0.97;
      
      // Opdater boldens position
      this.ballPosition += this.ballVelocity;

      // Sørg for at bolden ikke ryger gennem gulvet eller for højt op
      const glassHeight = 300; // Skal matche CSS
      if (this.ballPosition < 0) {
        this.ballPosition = 0;
        this.ballVelocity = 0;
      } else if (this.ballPosition > glassHeight + 50) {
        // Bolden er fløjet ud! Her kan man tilføje en "vundet"-tilstand
      }

      // Fortsæt til næste frame
      this.animationFrameId = requestAnimationFrame(this.gameLoop);
    }
  },
  beforeUnmount() {
    // Vigtigt at stoppe loop'et og lukke for lyden for at spare på ressourcer
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
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  background-color: #f0f0f0;
  text-align: center;
}

.glass-container {
  margin: 20px 0;
}

.glass {
  width: 150px;
  height: 300px;
  border: 8px solid #333;
  border-top: none;
  background-color: rgba(255, 255, 255, 0.5);
  border-radius: 0 0 20px 20px;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: flex-end; /* Bolden starter i bunden */
  overflow: hidden; /* Skjuler bolden når den er for høj */
}

.ball {
  /* Bruger en CSS variabel til at styre positionen for bedre ydeevne */
  --ball-bottom: 0px;
  position: absolute;
  bottom: var(--ball-bottom);
  width: 60px;
  height: 60px;
  background-color: #e74c3c;
  border-radius: 50%;
  transition: bottom 0.1s linear; /* Gør bevægelsen lidt blødere */
}

button {
  margin-top: 20px;
  padding: 12px 25px;
  font-size: 18px;
  cursor: pointer;
  border: none;
  background-color: #3498db;
  color: white;
  border-radius: 5px;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #2980b9;
}

.info {
  margin-top: 15px;
  color: #555;
}

.error {
  color: #c0392b;
  margin-top: 20px;
  max-width: 400px;
}
</style>