# 🏡 Cozy Room Physics

An interactive 2.5D cozy room built with **Vue.js 3**. This creative coding experiment features a custom physics-based swinging pendulum lamp on Canvas, real-time dynamic lighting, and a smooth day/night cycle.

## ✨ Features

- **Pendulum Physics:** Custom physics engine built with trigonometry (`Math.sin`, `Math.cos`, `Math.atan2`) to calculate gravity, inertia, and damping.
- **Dynamic Lighting:** Real-time lighting overlay mapped to the swinging lamp using `mix-blend-mode: multiply` and dynamic CSS radial gradients.
- **2.5D CSS Art:** The entire room (sofa, table, plant, window) is drawn using standard HTML `div`s and advanced CSS shadows/gradients without any images or SVGs.
- **Day/Night Cycle:** Click on the moon/sun to smoothly transition the environment between Night, Day, and Afternoon.
- **Interactive Light Switch:** A working switch on the wall that turns the lamp glow on and off.

## 🛠️ Tech Stack

- **Framework:** Vue 3 (Composition API)
- **Tooling:** Vite
- **Graphics:** `<canvas>` (for the lamp and light beam) + CSS3
- **Containerization:** Docker & Docker Compose

## 🚀 Getting Started (Step-by-Step)

You can run this project locally using either **Docker** (recommended) or standard **NPM**.

### Option 1: Running with Docker (Recommended)

1. **Clone the repository:**
   ```bash
   git clone git@github.com:matondojk/cozy-room-physics.git
   cd cozy-room-physics
   ```

2. **Build and start the container:**
   Make sure you have Docker and Docker Compose installed.
   ```bash
   docker-compose up --build
   ```

3. **Open in browser:**
   Go to [http://localhost:5173](http://localhost:5173). Any changes made to the code will automatically hot-reload!

### Option 2: Running with Node.js/NPM

1. **Clone the repository:**
   ```bash
   git clone git@github.com:matondojk/cozy-room-physics.git
   cd cozy-room-physics
   ```

2. **Install dependencies:**
   Make sure you have Node.js (v18+) installed.
   ```bash
   npm install
   ```

3. **Start the development server:**
   ```bash
   npm run dev
   ```

4. **Open in browser:**
   Go to [http://localhost:5173](http://localhost:5173).

## 🎮 How to Interact

- **Click and Drag** the lamp to pull the string back. Release it to watch the physics engine swing it to a rest.
- **Click the Sun/Moon** in the sky to change the time of day.
- **Click the Light Switch** on the left wall to turn the lamp on and off.

## 🧠 How it Works under the hood

The project elegantly avoids large external 2D libraries by combining DOM and Canvas:
1. **The Room (`<div class="room">`)** is styled with raw CSS to look 3D using inset shadows and `border` hacks.
2. **The Physics Layer (`<canvas>`)** sits on top of everything. It handles the `requestAnimationFrame` loop, reading the mouse coordinates to update the pendulum angle.
3. **The Light Overlay (`.lighting-overlay`)** is a `div` that covers the room with a `radial-gradient`. The JavaScript code updates the center of this gradient 60 times a second to match the X and Y coordinates of the Canvas lamp, producing incredibly cheap and smooth dynamic lighting via CSS's `mix-blend-mode: multiply`.

## 📄 License

This project is open-source and available under the MIT License. Feel free to clone, modify, and learn from it!
