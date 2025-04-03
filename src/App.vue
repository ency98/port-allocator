<!-- src/App.vue -->
<template>
  <div id="app">
    <header>
      <h1>Port Allocator</h1>
    </header>
    <router-view />
    <footer class="app-footer">
      <div class="footer-left">
        <button @click="toggleDebug" class="debug-toggle" :title="showDebug ? 'Hide Debug Info' : 'Show Debug Info'">
          <img src="/bug.svg" alt="Debug" class="debug-icon" :class="{ 'active': showDebug }">
        </button>
      </div>
      <div class="footer-right">
        <ThemeToggle />
      </div>
    </footer>
  </div>
</template>

<script>
import { ref } from 'vue';
import ThemeToggle from './components/ThemeToggle.vue';

export default {
  name: 'App',
  components: {
    ThemeToggle
  },
  setup() {
    const showDebug = ref(false);

    const toggleDebug = () => {
      showDebug.value = !showDebug.value;
      // Emit an event that the Home component can listen for
      window.dispatchEvent(new CustomEvent('toggle-debug', { 
        detail: { showDebug: showDebug.value } 
      }));
    };

    return {
      showDebug,
      toggleDebug
    };
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: var(--text-color, #2c3e50);
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  transition: color 0.3s ease, background-color 0.3s ease;
}

@media (max-width: 768px) {
  #app {
    padding: 15px;
  }
}

@media (max-width: 480px) {
  #app {
    padding: 10px;
  }
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 20px;
  border-bottom: 1px solid var(--bg3, #eee);
}

.app-footer {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid var(--bg3, #eee);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.footer-left, .footer-right {
  flex: 1;
}

.footer-left {
  display: flex;
  justify-content: flex-start;
}

.footer-right {
  display: flex;
  justify-content: flex-end;
}

.debug-toggle {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  border: none;
  background-color: var(--theme-toggle-bg, #3c3836);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
  padding: 0;
}

.debug-toggle:hover {
  transform: scale(1.1);
}

.debug-icon {
  width: 28px;
  height: 28px;
  filter: invert(100%) brightness(200%);
}

body.light-theme .debug-icon {
  filter: invert(10%);
}

.debug-icon.active {
  filter: invert(80%) sepia(30%) saturate(1000%) hue-rotate(340deg) brightness(100%);
}

body.light-theme .debug-icon.active {
  filter: invert(15%) sepia(90%) saturate(2500%) hue-rotate(340deg) brightness(90%);
}

@media (max-width: 768px) {
  .debug-toggle {
    width: 42px;
    height: 42px;
  }
  
  .debug-icon {
    width: 24px;
    height: 24px;
  }
}

@media (max-width: 480px) {
  .debug-toggle {
    width: 36px;
    height: 36px;
  }
  
  .debug-icon {
    width: 20px;
    height: 20px;
  }
}

button {
  background-color: var(--button-primary, #4CAF50);
  border: none;
  color: var(--button-text, white);
  padding: 10px 20px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 16px;
  margin: 4px 2px;
  cursor: pointer;
  border-radius: 4px;
  transition: background-color 0.3s ease;
}

button:hover {
  background-color: var(--button-primary-hover, #45a049);
  opacity: 0.9;
}
</style>