<template>
  <div>
    <Analytics />
    <NuxtPage />
  </div>
</template>

<script setup>
import { Analytics } from '@vercel/analytics/nuxt';

// Add global styles or meta tags for the entire application here
useHead({
  meta: [
    { name: 'robots', content: 'noindex, nofollow' },
    { name: 'viewport', content: 'width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no' }
  ]
});
</script>

<style>
:root {
  /* Color variables */
  --color-background: #121212;
  --color-surface: #1e1e1e;
  --color-primary: #2196f3;
  --color-primary-dark: #1976d2;
  --color-primary-light: #64b5f6;
  --color-primary-text: #ffffff;
  --color-secondary: #ff4081;
  --color-secondary-dark: #c60055;
  --color-secondary-text: #ffffff;
  --color-text-primary: #e0e0e0;
  --color-text-secondary: #a0a0a0;
  --color-text-disabled: #6c6c6c;
  --color-border: #333333;
  
  /* Semantic color variables */
  --green-3: #4caf50;
  --yellow-3: #fdd835;
  --red-3: #f44336;
  --gray-3: #9e9e9e;
  --gray-8: #424242;
  
  /* Spacing variables */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;
  
  /* Border radius */
  --border-radius-sm: 4px;
  --border-radius-md: 8px;
  --border-radius-lg: 12px;
  
  /* Shadows */
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.12), 0 1px 2px rgba(0, 0, 0, 0.24);
  --shadow-md: 0 3px 6px rgba(0, 0, 0, 0.16), 0 3px 6px rgba(0, 0, 0, 0.23);
  --shadow-lg: 0 10px 20px rgba(0, 0, 0, 0.19), 0 6px 6px rgba(0, 0, 0, 0.23);
}

/* Reset & base styles */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body {
  background-color: var(--color-background);
  color: var(--color-text-primary);
  font-family: Arial, sans-serif;
  height: 100%;
  overflow: hidden;
}

#__nuxt {
  height: 100%;
}
</style>