<template>
  <div class="grid-container">
    <div class="background-container">
      <div class="background" :class="{ active: isCurrentBackground }"></div>
      <div class="background" :class="{ active: !isCurrentBackground }"></div>
    </div>
    <div class="top-section"></div>

    <div class="middle-section">
      <button class="nav-button" @click="prevBackground">‹</button>
      <button
        :disabled="!isPlaying && allRecords.length != 0 ? false : true"
        @click="playRandomTrack"
        class="play-button"
        :style="{ boxShadow: boxShadow }"
      >
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
      trackTitle: "",
      trackArtist: "",
      boxShadow: "0 4px 10px #56ff086f",
      playedTracks: [],
      base: Airtable.base("app4KfOz94tUagKgW"),
      currentBackgroundIndex: 0,
      allRecords: [],
      nextTrack: null,
      loadedBackgrounds: [],
      isBackgroundChanging: false,
      isCurrentBackground: true,
      prevBackgroundIndex: 0,
    };
  },
  async mounted() {
    // Загружаем все фоны
    const backgrounds = await this.base("backgrounds")
      .select({ view: "back" })
      .all();

    if (backgrounds.length > 0) {
      // Предзагружаем все фоны
      this.loadedBackgrounds = await Promise.all(
        backgrounds.map(async (bg) => {
          const img = new Image();
          img.src = bg.fields.File[0].url;
          await img.decode();
          return img.src;
        })
      );

      this.currentBackgroundIndex = Math.floor(
        Math.random() * backgrounds.length
      );
      const background = document.querySelector(".background");
      background.style.backgroundImage = `url(${
        this.loadedBackgrounds[this.currentBackgroundIndex]
      })`;
    }

    // Загружаем все треки
    await this.loadAllRecords();
    await this.preloadNextTrack();
  },
  methods: {
    async loadAllRecords() {
      this.allRecords = await this.base("FILES")
        .select({ view: "musicInfo" })
        .all();
    },

    async changeBackground(direction) {
      if (this.isBackgroundChanging || this.loadedBackgrounds.length === 0)
        return;

      this.isBackgroundChanging = true;
      const backgrounds = document.querySelectorAll(".background");
      const currentBg = backgrounds[this.isCurrentBackground ? 0 : 1];
      const nextBg = backgrounds[this.isCurrentBackground ? 1 : 0];

      // Установка нового фона
      nextBg.style.backgroundImage = `url(${
        this.loadedBackgrounds[
          direction === "next"
            ? (this.currentBackgroundIndex + 1) % this.loadedBackgrounds.length
            : (this.currentBackgroundIndex -
                1 +
                this.loadedBackgrounds.length) %
              this.loadedBackgrounds.length
        ]
      })`;

      // Анимация сдвига
      currentBg.style.transform = `translateX(${
        direction === "next" ? "-100%" : "100%"
      })`;
      nextBg.style.transform = "translateX(0)";
      nextBg.style.opacity = 1;

      await new Promise((resolve) => setTimeout(resolve, 800));

      // Обновление индекса
      this.currentBackgroundIndex =
        direction === "next"
          ? (this.currentBackgroundIndex + 1) % this.loadedBackgrounds.length
          : (this.currentBackgroundIndex - 1 + this.loadedBackgrounds.length) %
            this.loadedBackgrounds.length;
      this.isCurrentBackground = !this.isCurrentBackground;
      this.isBackgroundChanging = false;

      // Управление слоями
      currentBg.style.zIndex = 0;
      nextBg.style.zIndex = 1;
    },

    async nextBackground() {
      await this.changeBackground("next");
    },

    async prevBackground() {
      await this.changeBackground("prev");
    },

    async preloadNextTrack() {
      try {
        const randomIndex = Math.floor(Math.random() * this.allRecords.length);
        const randomTrack = this.allRecords[randomIndex].fields;

        if (this.playedTracks.includes(randomTrack.NAME)) {
          if (this.playedTracks.length === this.allRecords.length) {
            this.playedTracks = [];
          } else {
            return this.preloadNextTrack();
          }
        }

        const response = await fetch(randomTrack.File[0].url);
        const blob = await response.blob();
        const objectURL = URL.createObjectURL(blob);
        this.nextTrack = {
          audio: new Audio(objectURL),
          title: randomTrack.NAME,
          artist: randomTrack.Artist,
          cover: randomTrack.Cover?.[0]?.url,
        };
      } catch (error) {
        console.error("Ошибка при прелоаде трека:", error);
      }
    },

    async playRandomTrack() {
      if (!this.nextTrack) return;

      try {
        // Используем предзагруженный трек
        this.trackTitle = this.nextTrack.title;
        this.trackArtist = this.nextTrack.artist;
        this.audio = this.nextTrack.audio;

        if (this.nextTrack.cover) {
          const background = document.querySelector(".play-button");
          background.style.backgroundImage = `url(${this.nextTrack.cover})`;
        }

        this.audio.play();
        this.isPlaying = true;
        this.playedTracks.push(this.trackTitle);

        // Сразу загружаем следующий трек
        await this.preloadNextTrack();

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
          if (this.nextTrack) {
            await this.playRandomTrack();
          }
        };
      } catch (error) {
        console.error("Ошибка:", error);
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
/* Все существующие стили остаются без изменений */
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
  transition: transform 0.8s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.8s ease-in-out;
}

.background:first-child {
  z-index: 1;
}

.background:last-child {
  z-index: 0;
}

.background-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: -1;
  background-image: url("../public/photo_2024-10-25_22-04-06.jpg");
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
</style>
