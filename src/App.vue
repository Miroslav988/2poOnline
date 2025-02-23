<template>
  <div class="grid-container">
    <div class="background"></div>
    <!-- Верхняя секция -->
    <div class="top-section"></div>

    <!-- Центральная секция -->
    <div class="middle-section">
      <button class="nav-button" @click="prevBackground">‹</button>
      <button
        :disabled="!isPlaying && allRecords.length != 0 ? false : true"
        @click="playRandomTrack"
        class="play-button"
        :style="{ boxShadow: boxShadow }"
      >
        <div v-if="isLoading" class="loader-container">
          <div class="loader"></div>
        </div>
        <div v-if="isPlaying" class="controls">
          <label for="volume"></label>
          <input
            type="range"
            id="volume"
            min="0"
            max="1"
            step="0.1"
            @input="setVolume"
            class="volume-slider"
          />
        </div>
      </button>
      <button class="nav-button" @click="nextBackground">›</button>
    </div>

    <!-- Нижняя секция -->
    <div class="bottom-section">
      <h3>{{ trackTitle }}</h3>
      <h3>{{ trackArtist }}</h3>
    </div>
  </div>
</template>

<script>
import Airtable from "airtable";

Airtable.configure({
  apiKey: import.meta.env.VITE_AIRTABLE_API_KEY,
  requestTimeout: 30000,
});

export default {
  name: "App",
  data() {
    return {
      audio: null,
      isPlaying: false,
      isLoading: false,
      trackTitle: "",
      trackArtist: "",
      boxShadow: "0 4px 10px #56ff086f",
      playedTracks: [],
      base: Airtable.base("app4KfOz94tUagKgW"),
      currentBackgroundIndex: 0,
      allRecords: [],
    };
  },
  async mounted() {
    const backgrounds = await this.base("backgrounds")
      .select({ view: "back" })
      .all();

    if (backgrounds.length > 0) {
      this.currentBackgroundIndex = Math.floor(
        Math.random() * backgrounds.length
      );
      const backgroundUrl =
        backgrounds[this.currentBackgroundIndex].fields.File[0].url;
      const background = document.querySelector(".background");
      background.style.backgroundImage = `url(${backgroundUrl})`;
    }

    // Загружаем все треки
    await this.loadAllRecords();
  },
  methods: {
    async loadAllRecords() {
      this.allRecords = await this.base("FILES")
        .select({ view: "musicInfo" })
        .all();
    },
    async nextBackground() {
      const backgrounds = await this.base("backgrounds")
        .select({ view: "back" })
        .all();
      if (backgrounds.length === 0) return;

      this.currentBackgroundIndex =
        (this.currentBackgroundIndex + 1) % backgrounds.length;
      const background = document.querySelector(".background");
      background.style.backgroundImage = `url(${
        backgrounds[this.currentBackgroundIndex].fields.File[0].url
      })`;
    },

    async prevBackground() {
      const backgrounds = await this.base("backgrounds")
        .select({ view: "back" })
        .all();

      if (backgrounds.length === 0) return;

      this.currentBackgroundIndex =
        (this.currentBackgroundIndex - 1 + backgrounds.length) %
        backgrounds.length;
      const background = document.querySelector(".background");
      background.style.backgroundImage = `url(${
        backgrounds[this.currentBackgroundIndex].fields.File[0].url
      })`;
    },
    async playRandomTrack() {
      this.isLoading = true;
      try {
        const randomIndex = Math.floor(Math.random() * this.allRecords.length);
        const randomTrack = this.allRecords[randomIndex].fields;

        if (this.playedTracks.includes(randomTrack.NAME)) {
          if (this.playedTracks.length === this.allRecords.length) {
            this.playedTracks = [];
          } else {
            return this.playRandomTrack();
          }
        }

        this.trackTitle = randomTrack.NAME;
        this.trackArtist = randomTrack.Artist;

        const response = await fetch(randomTrack.File[0].url);
        const blob = await response.blob();
        const objectURL = URL.createObjectURL(blob);
        this.audio = new Audio(objectURL);

        if (randomTrack.Cover && randomTrack.Cover[0]) {
          const background = document.querySelector(".play-button");
          background.style.backgroundImage = `url(${randomTrack.Cover[0].url})`;
        }

        this.audio.play();
        this.isPlaying = true;
        this.playedTracks.push(randomTrack.NAME);

        this.audioContext = new (window.AudioContext ||
          window.webkitAudioContext)();
        const source = this.audioContext.createMediaElementSource(this.audio);
        this.analyser = this.audioContext.createAnalyser();
        source.connect(this.analyser);
        this.analyser.connect(this.audioContext.destination);
        this.analyser.fftSize = 256;
        this.dataArray = new Uint8Array(this.analyser.frequencyBinCount);

        this.updateAnimation();
        this.audio.onended = async () => {
          this.isPlaying = false;
          try {
            await this.playRandomTrack();
          } catch (error) {
            console.error(
              "Ошибка при воспроизведении следующего трека:",
              error
            );
          }
        };
      } catch (error) {
        console.error("Ошибка:", error);
      } finally {
        this.isLoading = false;
      }
    },

    updateAnimation() {
      requestAnimationFrame(this.updateAnimation);
      this.analyser.getByteFrequencyData(this.dataArray);
      const average =
        this.dataArray.reduce((a, b) => a + b) / this.dataArray.length;
      this.boxShadow = `0 4px ${average * 1.3}px rgba(86, 255, 8, 0.63)`;
    },

    setVolume(event) {
      if (this.audio) {
        this.audio.volume = event.target.value;
      }
    },
  },
};
</script>

<style>
.grid-container {
  display: grid;
  grid-template-rows: 1fr 2fr 1fr;
  height: 100vh;
  width: 100vw;
}

.top-section {
  grid-row: 1;
  position: relative;
}

.middle-section {
  grid-row: 2;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.bottom-section {
  grid-row: 3;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: #56ff08;
  font-size: 1.5rem;
}

.background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-size: cover;
  background-position: center;
  z-index: -1;
}

.play-button {
  width: 300px;
  height: 300px;
  background-size: cover;
  background-position: center;
  cursor: pointer;
  background-image: url("../public/photo_2024-10-25_22-04-06.jpg");
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
}

.nav-button {
  padding: 2rem;
  background: none;
  border: none;
  color: #56ff08;
  font-size: 7rem;
  cursor: pointer;
  transition: opacity 0.3s;
}

.nav-button:hover {
  opacity: 0.7;
}

.volume-slider {
  margin-top: 1rem;
  width: 150px;
}

input[type="range"] {
  -webkit-appearance: none;
  background: #56ff08;
  height: 8px;
  border-radius: 5px;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  background: #56ff08;
  border-radius: 50%;
  cursor: pointer;
}

.loader-container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.loader {
  border: 4px solid rgba(86, 255, 8, 0.2);
  border-top: 4px solid #56ff08;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
