# Worldmap Project Guide

## Commands
- Development: `npm run dev` - Start development server
- Build: `npm run build` - Build for production
- Preview: `npm run preview` - Preview production build
- Generate: `npm run generate` - Generate static site
- Install: `npm install` - Install dependencies
- Prepare: `npm run postinstall` - Run Nuxt prepare

## Code Style Guidelines
- **Framework**: Nuxt.js 3.x with Vue 3.x
- **Formatting**: Follow Vue Single-File Component pattern
- **Naming**:
  - Components: PascalCase (e.g., `WorldMap.vue`)
  - Variables/Props: camelCase
  - Methods: camelCase with descriptive verbs
- **Imports**: Group by external/internal and order alphabetically
- **Types**: Use TypeScript interfaces/types for all components and functions
- **Error Handling**: Use try/catch with meaningful error messages
- **Components**: Prefer composition API with `<script setup>`