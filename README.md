# CARai Dashboard

A Vue 3-based dashboard application for visualizing and interacting with cellular data and machine learning models.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Development](#development)
- [Production Build](#production-build)
- [IDE Setup](#ide-setup)
- [Browser Extensions](#browser-extensions)
- [Configuration](#configuration)

## Overview

This project provides a modern web dashboard built with Vue 3 and Vite for cellular data visualization and analysis.

## Prerequisites

- [Node.js](https://nodejs.org/) (version 16 or higher recommended)
- [npm](https://www.npmjs.com/) or [yarn](https://yarnpkg.com/)
- [VS Code](https://code.visualstudio.com/) (recommended)

## Installation

Install project dependencies:

```sh
npm install
```

## Development

Start the development server with hot-reload:

```sh
npm run dev
```

The application will be available at `http://localhost:5173` (or another port if 5173 is in use).

### Alternative: Static HTML Development

For the static HTML dashboard (`dashboard.html`):

1. Install the [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) extension in VS Code
2. Right-click `dashboard.html` in the VS Code explorer
3. Select "Open with Live Server"

## Production Build

Create an optimized production build:

```sh
npm run build
```

The built files will be output to the `dist/` directory.

## IDE Setup

**Recommended:**

- [VS Code](https://code.visualstudio.com/)
- [Vue (Official)](https://marketplace.visualstudio.com/items?itemName=Vue.volar) extension
  - **Important:** Disable Vetur if you have it installed, as it conflicts with the official Vue extension

## Browser Extensions

**Chromium-based browsers** (Chrome, Edge, Brave):
- [Vue.js devtools](https://chromewebstore.google.com/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd)
- [Enable Custom Object Formatters](http://bit.ly/object-formatters) in DevTools

**Firefox:**
- [Vue.js devtools](https://addons.mozilla.org/en-US/firefox/addon/vue-js-devtools/)
- [Enable Custom Object Formatters](https://fxdx.dev/firefox-devtools-custom-object-formatters/) in DevTools

## Configuration

For advanced configuration options, see the [Vite Configuration Reference](https://vite.dev/config/).

