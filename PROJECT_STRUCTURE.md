# Comprehensive Project Architecture & Structure Guide

This project is a modern, high-performance web application built with **Next.js 14**, **Prismic CMS**, **Tailwind CSS**, and **Three.js (React Three Fiber)**. It uses a **Slice-based architecture** to separate content into modular, reusable components.

---

## 1. 📂 Detailed Directory Structure

```
├── .next/                   # Next.js build output (do not edit)
├── customtypes/             # Prismic Custom Type definitions (managed by Slice Machine)
│   └── page/                # Schema for "Page" documents
├── public/                  # Static assets served at root path
│   ├── fonts/               # Custom fonts (e.g., Alpino-Variable.woff2)
│   ├── hdr/                 # HDR Environment maps for 3D lighting (lobby.hdr)
│   ├── labels/              # Soda can texture labels (png)
│   ├── Soda-can.gltf        # 3D Model of the can
│   └── ...
├── src/
│   ├── app/                 # Next.js App Router
│   │   ├── [uid]/           # Dynamic Route Handler for Prismic pages
│   │   ├── api/             # API Endpoints (Preview, Exit Preview, Revalidate)
│   │   ├── layout.tsx       # Global Root Layout (Header, Footer, ViewCanvas)
│   │   ├── page.tsx         # Homepage Entry Point
│   │   └── slice-simulator/ # Route for local Slice Machine simulator
│   ├── components/          # Shared UI & 3D Components
│   │   ├── Header.tsx       # Site Navigation
│   │   ├── Footer.tsx       # Site Footer
│   │   ├── SodaCan.tsx      # 3D Model Component (loads .gltf + textures)
│   │   ├── FloatingCan.tsx  # Wrapper for SodCan with floating animation
│   │   ├── ViewCanvas.tsx   # The main R3F Canvas overlay
│   │   └── ...
│   ├── hooks/               # Custom Hooks
│   │   ├── useStore.ts      # Zustand state (manages 3D scene readiness)
│   │   └── ...
│   ├── slices/              # Prismic Slices (The core content modules)
│   │   ├── Hero/            # "Hero" Section Slice
│   │   ├── SkyDive/         # "SkyDive" Animation Slice
│   │   ├── Carousel/        # "Carousel" Interactive Slice
│   │   └── ...
│   └── prismicio.ts         # Prismic Client Config (Routes, Fetch Options)
├── .env.local               # Environment variables (API Keys)
├── next.config.mjs          # Next.js Config
├── postcss.config.mjs       # PostCSS/Tailwind Config
├── slicemachine.config.json # Prismic Slice Machine Sync Config
└── tailwind.config.js       # Tailwind Styling Config
```

---

## 2. 🏗️ Architecture & Core Concepts

### A. The "Slice" Pattern (CMS Integration)
Instead of building fixed templates (e.g., "About Page", "Contact Page"), this project uses **Slices**.
1.  **Definition**: A Slice is a horizontal section of a page (e.g., a Hero banner, a Text block, a Carousel).
2.  **Creation**: define Slices in **Slice Machine** (locally running UI).
3.  **Content**: Editors allow usage of these Slices in the Prismic Dashboard.
4.  **Rendering**: `src/app/page.tsx` fetches the page data and passes the list of slices to `<SliceZone />`, which matches them to the React components in `src/slices/`.

### B. 3D Rendering Strategy (ViewPortal Pattern)
To mix heavy 3D content with standard HTML scrolling, we use the **View Tracking** technique from `@react-three/drei`.

*   **Global Canvas (`ViewCanvas.tsx`)**:
    *   There is **ONE** single `<Canvas>` component in `src/app/layout.tsx`.
    *   It covers the entire screen (`position: fixed`, `pointer-events: none`).
    *   It stays persistent across page navigations.

*   **Local Views**:
    *   Inside a Slice (e.g., `Hero/index.tsx`), we render a `<View>` component.
    *   This component "tunnels" its children into the global Canvas.
    *   This allows us to place 3D objects "inside" specific HTML DOM elements while actually rendering them in the single efficient WebGL context.

### C. State Management (Zustand)
We use **Zustand** (`src/hooks/useStore.ts`) for global UI state.
*   **`ready` / `isReady`**:  Used to coordinate the loading of heavy 3D assets. The `Hero` slice animation waits for the 3D scene to report `isReady` before starting the GSAP entrance animation, ensuring no "pop-in" of unstyled content.

### D. Animation (GSAP)
**GreenSock Animation Platform (GSAP)** is used for complex choreographies.
*   **ScrollTrigger**: Animations that play based on scroll position (used in all slices).
*   **Timelines**: Sequencing animations (e.g., Text fades in -> Can floats up -> Background changes).
*   **Integration**: GSAP controls both DOM elements (HTML) and Three.js object properties (position/rotation) simultaneously.

---

## 3. 🧩 Key Component Deep Dive

### `src/components/SodaCan.tsx`
*   **Purpose**: The raw 3D model of the can.
*   **Data**: Loads `/Soda-can.gltf`.
*   **Dynamic Textures**: Accepts a `flavor` prop. It maps this flavor to a specific label texture (e.g., `labels/grape.png`) and applies it to the can's material.

### `src/components/FloatingCan.tsx`
*   **Purpose**: A wrapper around `SodaCan`.
*   **Behavior**: Uses `<Float>` from `@react-three/drei` to add idle floating/bobbing animation to the can without manual keyframing.

### `src/slices/Hero/index.tsx` & `Scene.tsx`
*   **Structure**: Split into `index.tsx` (all DOM/HTML/Text) and `Scene.tsx` (all 3D logic).
*   **Communication**: `index.tsx` renders the `<View>` that contains `<Scene />`.
*   **Animation**: `index.tsx` runs the GSAP ScrollTrigger that scrubs the background color of the `<body>` and animates text. `Scene.tsx` has its own GSAP animations for rotating the cans.

---

## 4. 🔄 Data Flow Summary

1.  **Request**: User visits `/`.
2.  **Server**: `src/app/page.tsx` runs on server.
3.  **Fetch**: calls `client.getByUID("page", "home")` to Prismic API.
4.  **Response**: Receives JSON with `title`, `meta`, and array of `slices`.
5.  **Render**:
    *   `RootLayout` renders `Header`, `ViewCanvas`, and `Footer`.
    *   `Page` component renders `<SliceZone>`.
    *   `SliceZone` iterates slices -> Renders `Hero`, then `SkyDive`, then `Carousel`.
    *   `Hero` mounts -> Mounts `<View>` (tunnels 3D to Canvas) -> Starts GSAP animations.

## 5. 🛠 Development Workflow

### Adding a New Feature
1.  **New Slice**: Run `npm run slicemachine`, create slice "Features".
2.  **Model**: Add fields (Title, Image, Repeatable Group) in Slice Machine.
3.  **Code**: Go to `src/slices/Features/index.tsx`. Scaffold the UI.
4.  **3D?**: If 3D is needed, import `<View>` and create a `Scene` component.
5.  **Simulate**: Check it in the Slice Simulator.
6.  **Push**: Push changes to Prismic.

