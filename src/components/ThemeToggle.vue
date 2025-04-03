<!-- src/components/ThemeToggle.vue -->
<template>
  <div class="theme-toggle">
    <button @click="toggleTheme" class="theme-toggle-button" :title="isDarkMode ? 'Switch to Light Mode' : 'Switch to Dark Mode'">
      <span v-if="isDarkMode" class="toggle-icon">☀️</span>
      <span v-else class="toggle-icon">🌙</span>
    </button>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue';

export default {
  name: 'ThemeToggle',
  setup() {
    const isDarkMode = ref(true); // Default to dark mode

    // Initialize theme based on localStorage or default to dark
    onMounted(() => {
      const savedTheme = localStorage.getItem('theme') || 'dark';
      isDarkMode.value = savedTheme === 'dark';
      applyTheme(isDarkMode.value ? 'dark' : 'light');
    });

    // Apply theme changes whenever isDarkMode changes
    watch(isDarkMode, (newValue) => {
      applyTheme(newValue ? 'dark' : 'light');
      localStorage.setItem('theme', newValue ? 'dark' : 'light');
    });

    // Apply theme by adding the appropriate class to the document body
    const applyTheme = (theme) => {
      if (theme === 'dark') {
        document.body.classList.add('dark-theme');
        document.body.classList.remove('light-theme');
      } else {
        document.body.classList.add('light-theme');
        document.body.classList.remove('dark-theme');
      }
    };

    // Toggle between light and dark mode
    const toggleTheme = () => {
      isDarkMode.value = !isDarkMode.value;
    };

    return {
      isDarkMode,
      toggleTheme
    };
  }
}
</script>

<style scoped>
.theme-toggle {
  position: relative;
  display: inline-flex;
  margin: 20px 0;
  justify-content: center;
}

.theme-toggle-button {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  border: none;
  background-color: var(--theme-toggle-bg, #3c3836);
  color: var(--fg, #fff);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
}

.theme-toggle-button:hover {
  transform: scale(1.1);
}

.toggle-icon {
  font-size: 20px;
}

@media (max-width: 768px) {
  .theme-toggle-button {
    width: 42px;
    height: 42px;
  }
  
  .toggle-icon {
    font-size: 18px;
  }
}

@media (max-width: 480px) {
  .theme-toggle-button {
    width: 36px;
    height: 36px;
  }
}
</style>