<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

const viewer = ref(null)
const fileInput = ref(null)
const canvasHost = ref(null)
const modelName = ref('Dumbo.glb')
const modelStatus = ref('Ready to explore')
const isLoading = ref(true)
const loadProgress = ref(0)
const loadError = ref('')
const autoRotate = ref(true)
const wireframe = ref(false)
const isFullscreen = ref(false)
const stats = ref({ meshes: 0, vertices: 0, materials: 0 })

let scene
let camera
let renderer
let controls
let frameId
let activeModel
let resizeObserver
let objectUrl
const materials = new Set()

const setWireframe = (enabled) => {
  if (!activeModel) return
  activeModel.traverse((child) => {
    if (child.isMesh && child.material) {
      const childMaterials = Array.isArray(child.material) ? child.material : [child.material]
      childMaterials.forEach((material) => { material.wireframe = enabled })
    }
  })
}

const frameModel = () => {
  if (!activeModel || !camera || !controls) return
  const bounds = new THREE.Box3().setFromObject(activeModel)
  const center = bounds.getCenter(new THREE.Vector3())
  const size = bounds.getSize(new THREE.Vector3())
  const maxSize = Math.max(size.x, size.y, size.z)
  const distance = maxSize / (2 * Math.tan((camera.fov * Math.PI) / 360))
  camera.position.set(center.x + distance * 0.8, center.y + distance * 0.48, center.z + distance * 1.1)
  camera.near = Math.max(maxSize / 1000, 0.01)
  camera.far = Math.max(maxSize * 100, 100)
  camera.updateProjectionMatrix()
  controls.target.copy(center)
  controls.minDistance = maxSize * 0.45
  controls.maxDistance = maxSize * 5
  controls.update()
}

const removeModel = () => {
  if (!activeModel) return
  scene.remove(activeModel)
  activeModel.traverse((child) => {
    if (child.isMesh) {
      child.geometry?.dispose()
      const childMaterials = Array.isArray(child.material) ? child.material : [child.material]
      childMaterials.forEach((material) => material.dispose())
    }
  })
  activeModel = null
}

const loadModel = (source, name) => {
  isLoading.value = true
  loadError.value = ''
  loadProgress.value = 0
  modelName.value = name
  modelStatus.value = 'Loading asset'
  const loader = new GLTFLoader()

  loader.load(source, (gltf) => {
    removeModel()
    activeModel = gltf.scene
    let meshCount = 0
    let vertexCount = 0
    materials.clear()
    activeModel.traverse((child) => {
      if (child.isMesh) {
        meshCount += 1
        vertexCount += child.geometry?.attributes?.position?.count || 0
        const childMaterials = Array.isArray(child.material) ? child.material : [child.material]
        childMaterials.forEach((material) => materials.add(material))
        child.castShadow = true
        child.receiveShadow = true
      }
    })
    scene.add(activeModel)
    stats.value = { meshes: meshCount, vertices: vertexCount, materials: materials.size }
    setWireframe(wireframe.value)
    frameModel()
    isLoading.value = false
    modelStatus.value = 'Ready to explore'
  }, (event) => {
    if (event.total) loadProgress.value = Math.round((event.loaded / event.total) * 100)
  }, (error) => {
    isLoading.value = false
    loadError.value = 'This file could not be loaded. Try another GLB asset.'
    modelStatus.value = 'Load failed'
    console.error(error)
  })
}

const chooseFile = () => fileInput.value?.click()
const handleFile = (event) => {
  const file = event.target.files?.[0]
  if (!file) return
  if (!file.name.toLowerCase().endsWith('.glb')) { loadError.value = 'Please choose a .glb file.'; return }
  if (objectUrl) URL.revokeObjectURL(objectUrl)
  objectUrl = URL.createObjectURL(file)
  loadModel(objectUrl, file.name)
  event.target.value = ''
}
const resetView = () => frameModel()
const toggleFullscreen = async () => {
  if (!document.fullscreenElement) await viewer.value?.requestFullscreen()
  else await document.exitFullscreen()
}
const syncFullscreen = () => { isFullscreen.value = Boolean(document.fullscreenElement) }
const animate = () => {
  frameId = requestAnimationFrame(animate)
  controls.autoRotate = autoRotate.value
  controls.update()
  renderer.render(scene, camera)
}
const resize = () => {
  if (!canvasHost.value) return
  const { clientWidth, clientHeight } = canvasHost.value
  camera.aspect = clientWidth / clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(clientWidth, clientHeight, false)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
}

onMounted(async () => {
  await nextTick()
  const width = canvasHost.value.clientWidth
  const height = canvasHost.value.clientHeight
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#e7e9e4')
  camera = new THREE.PerspectiveCamera(35, width / height, 0.01, 100)
  camera.position.set(4, 2.5, 5)
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.15
  canvasHost.value.appendChild(renderer.domElement)
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.autoRotateSpeed = 1.25
  controls.enablePan = false
  controls.minPolarAngle = 0.25
  controls.maxPolarAngle = Math.PI * 0.82
  scene.add(new THREE.HemisphereLight('#fffaf0', '#8d9691', 2.2))
  const keyLight = new THREE.DirectionalLight('#fff4dc', 3.5)
  keyLight.position.set(4, 7, 5)
  keyLight.castShadow = true
  scene.add(keyLight)
  const fillLight = new THREE.DirectionalLight('#c9e6e1', 2)
  fillLight.position.set(-5, 2, -4)
  scene.add(fillLight)
  const floor = new THREE.Mesh(new THREE.CircleGeometry(12, 64), new THREE.ShadowMaterial({ opacity: 0.16 }))
  floor.rotation.x = -Math.PI / 2
  floor.position.y = -1.2
  floor.receiveShadow = true
  scene.add(floor)
  resizeObserver = new ResizeObserver(resize)
  resizeObserver.observe(canvasHost.value)
  document.addEventListener('fullscreenchange', syncFullscreen)
  loadModel('/Dumbo.glb', 'Dumbo.glb')
  animate()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(frameId)
  resizeObserver?.disconnect()
  document.removeEventListener('fullscreenchange', syncFullscreen)
  if (objectUrl) URL.revokeObjectURL(objectUrl)
  renderer?.dispose()
})
</script>

<template>
  <main ref="viewer" class="app-shell">
    <header class="topbar">
      <a class="brand" href="/" aria-label="Object Studio home"><span class="brand-mark"><span></span><span></span><span></span></span><span>OBJECT<span class="brand-muted">/</span>STUDIO</span></a>
      <div class="topbar-meta"><span class="live-dot"></span> WebGL renderer online <span class="topbar-divider"></span> v1.0.0</div>
      <button class="upload-button" type="button" @click="chooseFile"><span class="plus-icon">+</span> Import GLB</button>
      <input ref="fileInput" class="visually-hidden" type="file" accept=".glb,model/gltf-binary" @change="handleFile" />
    </header>
    <section class="workspace">
      <aside class="sidebar">
        <div class="eyebrow">Asset workspace</div>
        <h1>Make objects<br /><em>feel real.</em></h1>
        <p class="intro">A quiet space to inspect, present, and understand your 3D work.</p>
        <div class="asset-card"><div class="asset-preview"><span class="asset-file">GLB</span><span class="asset-corner">↗</span></div><div class="asset-info"><strong>{{ modelName }}</strong><span>{{ modelStatus }}</span></div><span class="status-check" :class="{ loading: isLoading }">{{ isLoading ? `${loadProgress}%` : '✓' }}</span></div>
        <div class="section-label">Scene details</div>
        <dl class="details-list"><div><dt>Meshes</dt><dd>{{ stats.meshes || '—' }}</dd></div><div><dt>Vertices</dt><dd>{{ stats.vertices ? stats.vertices.toLocaleString() : '—' }}</dd></div><div><dt>Materials</dt><dd>{{ stats.materials || '—' }}</dd></div><div><dt>Format</dt><dd>Binary GLTF</dd></div></dl>
        <div class="sidebar-footer"><span class="footer-spark">✦</span><span>Drag to orbit<br />Scroll to zoom</span></div>
      </aside>
      <section class="stage-panel">
        <div class="stage-head"><span>01 / Presentation view</span><span class="stage-coordinate">X <b>0.00</b> &nbsp; Y <b>0.00</b> &nbsp; Z <b>0.00</b></span></div>
        <div ref="canvasHost" class="canvas-host"><div class="stage-grid"></div><div v-if="isLoading" class="loading-state"><span class="loader-ring"></span><span>Preparing your object {{ loadProgress }}%</span></div><div v-if="loadError" class="error-state"><strong>{{ loadError }}</strong><button type="button" @click="chooseFile">Choose another file</button></div><div class="canvas-badge">DRAG TO EXPLORE <span>↗</span></div><div class="axis-widget"><span class="axis-y">Y</span><span class="axis-x">X</span><span class="axis-z">Z</span><i></i></div></div>
        <div class="stage-toolbar"><div class="toolbar-group"><button class="tool-button" :class="{ active: autoRotate }" type="button" title="Toggle automatic rotation" @click="autoRotate = !autoRotate"><span class="rotate-icon">↻</span><span>Auto rotate</span></button><button class="tool-button" :class="{ active: wireframe }" type="button" title="Toggle wireframe" @click="wireframe = !wireframe; setWireframe(wireframe)"><span class="wire-icon">◇</span><span>Wireframe</span></button></div><div class="toolbar-group"><button class="icon-button" type="button" title="Reset camera" @click="resetView">⟲</button><button class="icon-button" type="button" :title="isFullscreen ? 'Exit fullscreen' : 'Enter fullscreen'" @click="toggleFullscreen">{{ isFullscreen ? '×' : '⛶' }}</button></div></div>
      </section>
    </section>
    <footer class="page-footer"><span>OBJECT STUDIO <b>•</b> 3D ASSET VIEWER</span><span>Built for the details that matter.</span><span>© 2026</span></footer>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=Manrope:wght@400;500;600;700;800&display=swap');
:root { font-family: 'Manrope', sans-serif; color: #1e2725; background: #f5f5f1; font-synthesis: none; } * { box-sizing: border-box; } body { margin: 0; min-width: 320px; } button { font: inherit; }
.app-shell { min-height: 100vh; display: flex; flex-direction: column; background: #f5f5f1; }.topbar { height: 78px; display: flex; align-items: center; gap: 28px; padding: 0 4.2vw; border-bottom: 1px solid #daddd8; }.brand { display: flex; align-items: center; gap: 11px; color: #202a27; font-size: 11px; font-weight: 800; letter-spacing: .14em; text-decoration: none; }.brand-muted { color: #9da6a1; padding: 0 2px; }.brand-mark { display: flex; align-items: end; gap: 3px; height: 18px; }.brand-mark span { display: block; width: 4px; background: #d2df52; }.brand-mark span:nth-child(1) { height: 9px; }.brand-mark span:nth-child(2) { height: 14px; }.brand-mark span:nth-child(3) { height: 18px; }.topbar-meta { margin-left: auto; color: #89918c; font: 10px 'DM Mono', monospace; letter-spacing: .06em; text-transform: uppercase; }.live-dot { display: inline-block; width: 6px; height: 6px; margin-right: 7px; border-radius: 50%; background: #a7c536; box-shadow: 0 0 0 4px #e7eed1; }.topbar-divider { display: inline-block; width: 1px; height: 13px; margin: 0 14px -3px; background: #d2d6d1; }.upload-button { border: 0; padding: 12px 17px; color: #f8faf2; background: #273331; cursor: pointer; font-size: 11px; font-weight: 700; }.plus-icon { margin-right: 8px; color: #d2df52; font-size: 17px; vertical-align: -1px; }
.workspace { flex: 1; display: grid; grid-template-columns: minmax(260px, 28%) 1fr; min-height: 680px; padding: 4.5vw 4.2vw 3vw; gap: 5vw; }.sidebar { display: flex; flex-direction: column; max-width: 330px; padding: 17px 0 0; }.eyebrow, .section-label, .stage-head, .page-footer { color: #8e9892; font: 10px 'DM Mono', monospace; letter-spacing: .1em; text-transform: uppercase; }.eyebrow { color: #a7b834; }.sidebar h1 { margin: 21px 0 17px; color: #26322e; font-size: clamp(40px, 4.7vw, 68px); line-height: .98; letter-spacing: -.065em; }.sidebar h1 em { color: #a2af31; font-style: normal; }.intro { max-width: 260px; margin: 0 0 47px; color: #7b8580; font-size: 13px; line-height: 1.8; }.asset-card { display: flex; align-items: center; gap: 12px; padding: 10px; background: #fff; border: 1px solid #e3e5df; box-shadow: 0 7px 24px #313b2412; }.asset-preview { position: relative; display: grid; place-items: center; width: 48px; height: 48px; color: #fff; background: #303d38; font: 9px 'DM Mono', monospace; }.asset-file { padding: 4px 3px; border: 1px solid #89968d; }.asset-corner { position: absolute; top: 4px; right: 5px; color: #cad857; }.asset-info { display: flex; flex: 1; flex-direction: column; gap: 5px; min-width: 0; }.asset-info strong { overflow: hidden; color: #36413d; font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }.asset-info span { color: #9ca49f; font: 10px 'DM Mono', monospace; }.status-check { display: grid; place-items: center; width: 20px; height: 20px; color: #99ad2d; border: 1px solid #d8e1a1; border-radius: 50%; font: 11px 'DM Mono', monospace; }.status-check.loading { width: auto; height: auto; padding: 3px; border: 0; color: #9da69f; font-size: 9px; }.section-label { margin: 48px 0 15px; color: #afb6b0; }.details-list { margin: 0; border-top: 1px solid #dfe2dc; }.details-list div { display: flex; justify-content: space-between; padding: 11px 0; border-bottom: 1px solid #dfe2dc; }.details-list dt { color: #8b948e; font-size: 11px; }.details-list dd { margin: 0; color: #4e5c55; font: 10px 'DM Mono', monospace; }.sidebar-footer { display: flex; align-items: center; gap: 12px; margin-top: auto; color: #9da49e; font: 10px 'DM Mono', monospace; line-height: 1.7; }.footer-spark { color: #c3d34e; font-size: 24px; }
.stage-panel { min-width: 0; display: flex; flex-direction: column; }.stage-head { display: flex; justify-content: space-between; padding: 8px 0 14px; border-bottom: 1px solid #dfe2dc; }.stage-coordinate { color: #aab1ac; }.stage-coordinate b { color: #69756e; font-weight: 400; }.canvas-host { position: relative; flex: 1; min-height: 510px; overflow: hidden; background: #e7e9e4; }.canvas-host canvas { position: relative; z-index: 1; display: block; width: 100%; height: 100%; }.stage-grid { position: absolute; inset: 0; z-index: 0; opacity: .32; background-image: linear-gradient(#c6ccc6 1px, transparent 1px), linear-gradient(90deg, #c6ccc6 1px, transparent 1px); background-size: 60px 60px; mask-image: linear-gradient(to bottom, transparent, #000 40%, transparent); }.canvas-badge { position: absolute; z-index: 2; right: 24px; bottom: 23px; padding: 8px 10px; color: #a2aaa4; border: 1px solid #cbd1cc; background: #e7e9e4cc; font: 9px 'DM Mono', monospace; letter-spacing: .08em; }.canvas-badge span { margin-left: 11px; color: #9caf30; }.axis-widget { position: absolute; z-index: 2; right: 23px; top: 23px; width: 43px; height: 43px; border: 1px solid #c9d0ca; border-radius: 50%; color: #919b95; font: 8px 'DM Mono', monospace; }.axis-widget i { position: absolute; left: 21px; top: 8px; width: 1px; height: 26px; background: #b6c0b9; }.axis-widget span { position: absolute; }.axis-y { top: 3px; left: 19px; color: #b1c33e; }.axis-x { bottom: 8px; left: 31px; }.axis-z { bottom: 8px; left: 8px; }.loading-state, .error-state { position: absolute; z-index: 4; inset: 0; display: grid; place-content: center; justify-items: center; gap: 13px; color: #738078; font: 10px 'DM Mono', monospace; }.loader-ring { width: 30px; height: 30px; border: 2px solid #cad2cc; border-top-color: #aabd35; border-radius: 50%; animation: spin 1s linear infinite; }.error-state { color: #65716b; text-align: center; font-family: 'Manrope', sans-serif; }.error-state button { padding: 9px 12px; border: 1px solid #b5c16a; color: #697722; background: transparent; cursor: pointer; font-size: 11px; }.stage-toolbar { display: flex; justify-content: space-between; padding-top: 13px; }.toolbar-group { display: flex; gap: 5px; }.tool-button, .icon-button { display: flex; align-items: center; gap: 8px; border: 1px solid transparent; color: #8e9992; background: transparent; cursor: pointer; font-size: 10px; }.tool-button { padding: 7px 9px; }.tool-button.active { color: #52631d; border-color: #d6dfa7; background: #f1f5df; }.rotate-icon, .wire-icon { color: #a7b837; font-size: 18px; }.wire-icon { font-size: 16px; }.icon-button { justify-content: center; width: 32px; height: 30px; border-color: #d6dad4; font-size: 18px; }.icon-button:hover, .tool-button:hover { border-color: #b8c877; color: #596a23; }.page-footer { display: flex; justify-content: space-between; padding: 0 4.2vw 25px; font-size: 9px; }.page-footer b { color: #c4d451; padding: 0 8px; }.page-footer span:nth-child(2) { color: #b1b7b1; text-transform: none; letter-spacing: 0; }.visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }
@keyframes spin { to { transform: rotate(360deg); } } @media (max-width: 760px) { .topbar { height: 65px; padding: 0 20px; }.topbar-meta { display: none; }.upload-button { margin-left: auto; }.workspace { display: block; min-height: auto; padding: 34px 20px 42px; }.sidebar { max-width: none; padding: 0; }.sidebar h1 { margin-top: 16px; font-size: 48px; }.intro { margin-bottom: 28px; }.section-label { margin-top: 30px; }.sidebar-footer { display: none; }.stage-panel { margin-top: 42px; }.stage-coordinate { display: none; }.canvas-host { min-height: 62vh; }.page-footer { padding: 0 20px 20px; font-size: 8px; }.page-footer span:nth-child(2) { display: none; } }
</style>
