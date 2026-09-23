# Object Studio 3D Viewer

This project is a Vue 3 and Vite application for displaying and inspecting GLB 3D models in the browser with Three.js.

The application currently:

- Loads `public/Dumbo.glb` by default.
- Allows users to orbit around the model and zoom in or out.
- Supports automatic rotation.
- Supports wireframe mode.
- Allows users to import another local `.glb` file.
- Displays mesh, vertex, material, and format information.
- Supports camera reset and fullscreen mode.
- Provides loading, progress, and error states.
- Adapts its layout for desktop and mobile screens.

## Project Structure

```text
3D_showcase_room/
|-- index.html
|-- package.json
|-- package-lock.json
|-- jsconfig.json
|-- vite.config.js
|-- README.md
|-- PROJECT_DOCUMENTATION.md
|-- public/
|   |-- Dumbo.glb
|   `-- favicon.ico
|-- src/
|   |-- App.vue
|   `-- main.js
`-- .vscode/
    |-- extensions.json
    `-- settings.json
```

Generated folders such as `node_modules/` and `dist/` are not source code. They are excluded from version control by `.gitignore`.

## Application Entry Point

### `index.html`

This is the browser document. It defines:

- The document language.
- The favicon.
- The responsive viewport.
- The browser tab title.
- The `#app` element where Vue is mounted.
- The module script that loads `src/main.js`.

```html
<div id="app"></div>
<script type="module" src="/src/main.js"></script>
```

### `src/main.js`

This file bootstraps Vue:

```js
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

It imports the root component and mounts it into the `#app` element from `index.html`.

## Main Component

### `src/App.vue`

`App.vue` contains the main application logic, page markup, and styling. It uses Vue's `<script setup>` syntax.

The component has three main responsibilities:

1. Manage reactive UI state with Vue.
2. Create and control the Three.js scene.
3. Render the Object Studio interface around the 3D canvas.

## Vue State

The component uses Vue `ref()` values for data that needs to update the interface:

```js
const modelName = ref('Dumbo.glb')
const modelStatus = ref('Ready to explore')
const isLoading = ref(true)
const loadProgress = ref(0)
const loadError = ref('')
const autoRotate = ref(true)
const wireframe = ref(false)
const isFullscreen = ref(false)
const stats = ref({ meshes: 0, vertices: 0, materials: 0 })
```

The template reads these values with expressions such as:

```html
{{ modelName }}
{{ modelStatus }}
{{ stats.meshes }}
```

When a value changes, Vue updates the corresponding part of the page automatically.

Three.js objects such as the scene, camera, renderer, and controls are stored in regular variables because they do not need to trigger Vue template updates.

## Three.js Scene Setup

The scene is created inside the `onMounted()` lifecycle hook. This ensures that the canvas host element already exists before Three.js tries to use it.

The setup creates:

- A `THREE.Scene`.
- A `THREE.PerspectiveCamera`.
- A `THREE.WebGLRenderer`.
- `OrbitControls`.
- A hemisphere light.
- A warm directional key light.
- A cool directional fill light.
- A shadow-receiving floor.

The renderer is configured with:

- Antialiasing.
- Shadows.
- sRGB output color space.
- ACES filmic tone mapping.
- A capped device pixel ratio of 2 for performance.

The scene background is a light gray-green color that matches the surrounding interface.

## Rendering Loop

The `animate()` function creates the continuous render loop:

```js
const animate = () => {
  frameId = requestAnimationFrame(animate)
  controls.autoRotate = autoRotate.value
  controls.update()
  renderer.render(scene, camera)
}
```

Every frame:

1. The browser schedules the next frame.
2. Orbit controls update their damping and rotation state.
3. Automatic rotation is enabled or disabled based on Vue state.
4. Three.js renders the scene from the current camera position.

## GLB Loading

The project uses Three.js's `GLTFLoader` to load binary GLTF files.

The default model is loaded with:

```js
loadModel('/Dumbo.glb', 'Dumbo.glb')
```

Because `Dumbo.glb` is inside the `public/` directory, Vite serves it from the root URL:

```text
public/Dumbo.glb -> /Dumbo.glb
```

When the model finishes loading, the application:

1. Removes the previous model, if one exists.
2. Stores the loaded scene in `activeModel`.
3. Traverses all child objects.
4. Counts meshes.
5. Counts vertices from position attributes.
6. Collects unique materials.
7. Enables shadow casting and receiving on meshes.
8. Adds the model to the Three.js scene.
9. Updates the sidebar statistics.
10. Applies the current wireframe setting.
11. Frames the camera around the model.
12. Changes the status to `Ready to explore`.

Loading progress is calculated from the loader event's `loaded` and `total` byte counts.

If loading fails, the interface displays an error message and allows the user to choose another file.

## Automatic Camera Framing

The `frameModel()` function keeps models at a useful distance from the camera.

It creates a bounding box with:

```js
const bounds = new THREE.Box3().setFromObject(activeModel)
```

It then calculates:

- The center of the model.
- The width, height, and depth of the model.
- The largest dimension.
- A suitable camera distance based on the camera field of view.

The camera is positioned relative to the model center, and the orbit controls target is moved to that center. Minimum and maximum zoom distances are also calculated from the model size.

This allows models with different dimensions to be displayed correctly without manually configuring each asset.

## Orbit Controls

`OrbitControls` provides mouse, touch, and scroll interaction.

The current settings include:

```js
controls.enableDamping = true
controls.enablePan = false
controls.autoRotateSpeed = 1.25
controls.minPolarAngle = 0.25
controls.maxPolarAngle = Math.PI * 0.82
```

This means users can rotate and zoom the object, but cannot pan it away from the stage. The vertical camera angle is also restricted so the view remains controlled.

## Importing a Local GLB

The visible `Import GLB` button triggers a hidden file input:

```js
const chooseFile = () => fileInput.value?.click()
```

The input accepts binary GLTF files:

```html
<input
  type="file"
  accept=".glb,model/gltf-binary"
  @change="handleFile"
/>
```

When a file is selected:

1. The file extension is checked.
2. A temporary browser object URL is created with `URL.createObjectURL()`.
3. The object URL is passed to `GLTFLoader`.
4. The selected filename appears in the asset card.
5. The previous object URL is released with `URL.revokeObjectURL()` when replaced.

The file stays local to the browser. It is not uploaded to a server.

## Wireframe Mode

The `setWireframe()` function traverses every mesh in the loaded object and sets the `wireframe` property on each material.

It supports both:

- A single material.
- An array of materials.

The toolbar button updates the Vue state and applies the new setting immediately.

## Fullscreen Mode

The fullscreen button uses the browser Fullscreen API:

```js
viewer.value?.requestFullscreen()
```

When the browser emits `fullscreenchange`, the component updates `isFullscreen` so the button label stays synchronized with the browser state.

Fullscreen CSS hides the regular header and footer and gives the viewer the available viewport.

## Responsive Layout

On larger screens, the workspace uses two columns:

- A left sidebar for asset information and statistics.
- A right presentation stage for the Three.js canvas.

At widths below `760px`, the layout changes to a vertical flow:

- The sidebar appears first.
- The 3D stage appears below it.
- Secondary coordinate information is hidden.
- The stage receives a viewport-based minimum height.
- Footer content is reduced.

All application styling is currently contained in `src/App.vue`.

The interface uses:

- `Manrope` for general UI text.
- `DM Mono` for technical labels and metadata.
- A light gray-green background.
- Dark green interface text.
- Lime accents for active controls and status indicators.

## Cleanup

The `onBeforeUnmount()` hook handles cleanup by:

- Cancelling the animation frame.
- Disconnecting the resize observer.
- Removing the fullscreen event listener.
- Revoking temporary uploaded-file URLs.
- Disposing the WebGL renderer.

This prevents the viewer from continuing to render after the Vue component is removed.

## Resizing

A `ResizeObserver` watches the canvas host element. When its dimensions change, the `resize()` function:

1. Updates the camera aspect ratio.
2. Updates the camera projection matrix.
3. Resizes the renderer.
4. Reapplies the capped device pixel ratio.

This keeps the model and canvas correctly proportioned when the browser window or layout changes.

## Vite Configuration

### `vite.config.js`

Vite is configured with:

```js
plugins: [
  vue(),
  vueDevTools(),
]
```

The Vue plugin allows Vite to compile `.vue` files. The Vue DevTools plugin adds development tooling during local development.

The `@` alias points to the `src` directory, allowing imports such as:

```js
import Component from '@/components/Component.vue'
```

## Dependencies

The main dependencies are declared in `package.json`.

### Runtime dependencies

- `vue`: UI framework.
- `three`: WebGL and 3D rendering engine.
- `@tresjs/core`: Vue integration utilities for Three.js.
- `@tresjs/cientos`: additional Tres utilities.

The current implementation directly uses Three.js. The Tres packages are installed but are not currently imported by `App.vue`.

### Development dependencies

- `vite`: development server and production bundler.
- `@vitejs/plugin-vue`: Vue support for Vite.
- `vite-plugin-vue-devtools`: Vue debugging tools.

The project expects Node.js 22 or Node.js 24 and newer.

## Supporting Files

### `public/Dumbo.glb`

The default 3D asset. Files in `public/` are served directly from the site root and copied into the production output.

### `public/favicon.ico`

The browser tab icon.

### `jsconfig.json`

Defines the `@/*` path alias and excludes `node_modules` and `dist` from JavaScript tooling.

### `.gitignore`

Excludes generated and local files such as:

- `node_modules/`
- `dist/`
- Logs.
- Coverage output.
- Local editor files.
- Test screenshots.

### `.vscode/extensions.json`

Recommends the Vue Official VS Code extension.

### `.vscode/settings.json`

Configures VS Code file nesting so related configuration files appear grouped together in the Explorer.

### `README.md`

Contains the original Vue and Vite setup instructions. This document contains the application-specific documentation.

## Running the Project

From the project directory:

```sh
npm install
npm run dev
```

Open the local URL shown by Vite in a browser.

To create a production build:

```sh
npm run build
```

To preview the production build locally:

```sh
npm run preview
```

## Runtime Flow

```text
index.html
  -> src/main.js
    -> App.vue mounts
      -> Three.js scene is created
      -> Lights, controls, camera, and floor are added
      -> Dumbo.glb is loaded
      -> Model statistics are calculated
      -> Camera frames the model
      -> requestAnimationFrame renders continuously
      -> Vue controls update the viewer
```

The project is intentionally compact. `App.vue` currently owns the page UI, viewer state, Three.js scene, model loading, and styles. A larger production application could split these responsibilities into separate components and composables, but the current structure keeps the prototype easy to follow.
