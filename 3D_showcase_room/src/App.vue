<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

const viewer = ref(null)
const fileInput = ref(null)
const canvasHost = ref(null)
const currentPage = ref(window.location.hash === '#viewer' ? 'viewer' : window.location.hash === '#blank' ? 'blank' : 'menu')
const modelName = ref('Dumbo.glb')
const modelStatus = ref('Ready to explore')
const isLoading = ref(true)
const loadProgress = ref(0)
const loadError = ref('')
const autoRotate = ref(true)
const wireframe = ref(false)
const isFullscreen = ref(false)
const stats = ref({ meshes: 0, vertices: 0, materials: 0 })
const modelFileSize = ref('Loading...')
const hoverCard = ref({ visible: false, x: 0, y: 0, partName: '' })

let scene
let camera
let renderer
let controls
let frameId
let activeModel
let resizeObserver
let objectUrl
const raycaster = new THREE.Raycaster()
const pointer = new THREE.Vector2()
const materials = new Set()

const formatFileSize = (bytes) => {
  if (!bytes) return 'Unknown size'
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(2)} MB`
}

const updateHoverCard = (event) => {
  if (!activeModel || !camera || !canvasHost.value || isLoading.value) {
    hoverCard.value.visible = false
    return
  }

  const bounds = canvasHost.value.getBoundingClientRect()
  pointer.x = ((event.clientX - bounds.left) / bounds.width) * 2 - 1
  pointer.y = -((event.clientY - bounds.top) / bounds.height) * 2 + 1
  raycaster.setFromCamera(pointer, camera)
  const hit = raycaster.intersectObject(activeModel, true)[0]

  if (!hit) {
    hoverCard.value.visible = false
    return
  }

  const cardWidth = 190
  const cardHeight = 112
  const offset = 18
  hoverCard.value = {
    visible: true,
    x: Math.min(event.clientX - bounds.left + offset, bounds.width - cardWidth - 12),
    y: Math.min(event.clientY - bounds.top + offset, bounds.height - cardHeight - 12),
    partName: hit.object.name || 'Main object',
  }
}

const hideHoverCard = () => {
  hoverCard.value.visible = false
}

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
  modelFileSize.value = formatFileSize(file.size)
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
const handleHashChange = () => window.location.reload()

onMounted(async () => {
  window.addEventListener('hashchange', handleHashChange)
  if (currentPage.value !== 'viewer') return
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
  fetch('/Dumbo.glb', { method: 'HEAD' }).then((response) => {
    const size = response.headers.get('content-length')
    if (size) modelFileSize.value = formatFileSize(Number(size))
  }).catch(() => { modelFileSize.value = 'Unknown size' })
  loadModel('/Dumbo.glb', 'Dumbo.glb')
  animate()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(frameId)
  window.removeEventListener('hashchange', handleHashChange)
  resizeObserver?.disconnect()
  document.removeEventListener('fullscreenchange', syncFullscreen)
  if (objectUrl) URL.revokeObjectURL(objectUrl)
  renderer?.dispose()
})
</script>

<template>
  <main v-if="currentPage === 'menu'" class="menu-shell">
    <div class="menu-orbit menu-orbit-one"></div>
    <div class="menu-orbit menu-orbit-two"></div>
    <header class="menu-header">
      <a class="brand" href="#menu" aria-label="Object Studio home"><span class="brand-mark"><span></span><span></span><span></span></span><span>OBJECT<span class="brand-muted">/</span>STUDIO</span></a>
      <span class="menu-kicker">Workspace navigator</span>
    </header>
    <section class="menu-content">
      <div class="menu-intro">
        <span class="menu-eyebrow">Choose a destination</span>
        <h1>Where should<br /><em>we go next?</em></h1>
        <p>Open a focused space for your 3D work.</p>
      </div>
      <nav class="destination-grid" aria-label="Main destinations">
        <a class="destination-card destination-card-primary" href="#viewer">
          <span class="destination-index">01</span>
          <span class="destination-icon destination-icon-cube"><i></i></span>
          <span class="destination-copy"><strong>GLB visualization</strong><small>Inspect and present a 3D asset</small></span>
          <span class="destination-arrow">↗</span>
        </a>
        <a class="destination-card" href="#blank">
          <span class="destination-index">02</span>
          <span class="destination-icon destination-icon-empty"><i></i></span>
          <span class="destination-copy"><strong>New workspace</strong><small>A blank space for what comes next</small></span>
          <span class="destination-arrow">↗</span>
        </a>
      </nav>
    </section>
    <footer class="menu-footer"><span>OBJECT STUDIO <b>•</b> 3D ASSET VIEWER</span><span>© 2026</span></footer>
  </main>
  <main v-else-if="currentPage === 'blank'" class="blank-shell">
    <header class="menu-header">
      <a class="brand" href="#menu" aria-label="Object Studio home"><span class="brand-mark"><span></span><span></span><span></span></span><span>OBJECT<span class="brand-muted">/</span>STUDIO</span></a>
      <a class="back-link" href="#menu">← Back to destinations</a>
    </header>
    <section class="blank-content"><span class="menu-eyebrow">Workspace 02</span><h1>A blank canvas<br /><em>awaits.</em></h1><p>This space is ready for the next experience.</p></section>
  </main>
  <main v-else ref="viewer" class="app-shell">
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
        <div ref="canvasHost" class="canvas-host" @pointermove="updateHoverCard" @pointerleave="hideHoverCard"><div class="stage-grid"></div><div v-if="isLoading" class="loading-state"><span class="loader-ring"></span><span>Preparing your object {{ loadProgress }}%</span></div><div v-if="loadError" class="error-state"><strong>{{ loadError }}</strong><button type="button" @click="chooseFile">Choose another file</button></div><div v-if="hoverCard.visible" class="hover-card" :style="{ left: `${hoverCard.x}px`, top: `${hoverCard.y}px` }"><div class="hover-card-kicker">Selected object</div><strong>{{ modelName }}</strong><div class="hover-card-part">{{ hoverCard.partName }}</div><dl><div><dt>File size</dt><dd>{{ modelFileSize }}</dd></div><div><dt>Meshes</dt><dd>{{ stats.meshes }}</dd></div></dl></div><div class="canvas-badge">DRAG TO EXPLORE <span>↗</span></div><div class="axis-widget"><span class="axis-y">Y</span><span class="axis-x">X</span><span class="axis-z">Z</span><i></i></div></div>
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
.menu-shell, .blank-shell { position: relative; min-height: 100vh; overflow: hidden; background: #f1f3ed; }.menu-header { position: relative; z-index: 1; display: flex; align-items: center; justify-content: space-between; height: 86px; padding: 0 6vw; border-bottom: 1px solid #d9ded5; }.menu-kicker, .back-link { color: #849089; font: 10px 'DM Mono', monospace; letter-spacing: .08em; text-transform: uppercase; }.back-link { color: #4d5d54; text-decoration: none; }.menu-content { position: relative; z-index: 1; max-width: 1080px; margin: 0 auto; padding: 13vh 6vw 11vh; }.menu-intro { max-width: 600px; }.menu-eyebrow { color: #9aaa2f; font: 10px 'DM Mono', monospace; letter-spacing: .12em; text-transform: uppercase; }.menu-intro h1, .blank-content h1 { margin: 22px 0 18px; color: #26342f; font-size: clamp(48px, 7vw, 94px); line-height: .94; letter-spacing: -.07em; }.menu-intro h1 em, .blank-content h1 em { color: #9aaa2f; font-style: normal; }.menu-intro p, .blank-content p { color: #78857d; font-size: 14px; line-height: 1.7; }.destination-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 18px; margin-top: 10vh; }.destination-card { position: relative; display: flex; flex-direction: column; min-height: 260px; padding: 26px; color: #34433c; border: 1px solid #d4dbd1; background: #e6eae2; text-decoration: none; transition: transform .25s ease, background .25s ease, border-color .25s ease; }.destination-card:hover { transform: translateY(-6px); border-color: #a6b43d; background: #eef1e9; }.destination-card-primary { color: #eef3e4; border-color: #35443d; background: #35443d; }.destination-card-primary:hover { border-color: #35443d; background: #405148; }.destination-index { color: #94a232; font: 11px 'DM Mono', monospace; }.destination-card-primary .destination-index { color: #d2df52; }.destination-icon { position: absolute; top: 25px; right: 28px; width: 42px; height: 42px; border: 1px solid #aeb8ac; }.destination-icon-cube { transform: rotate(30deg) skewY(-10deg); border-color: #b8c65a; }.destination-icon-cube i { position: absolute; inset: 8px; border: 1px solid #b8c65a; }.destination-icon-empty { border-style: dashed; }.destination-icon-empty i { position: absolute; left: 10px; top: 20px; width: 20px; border-top: 1px solid #95a296; }.destination-copy { display: flex; flex-direction: column; gap: 8px; margin-top: auto; }.destination-copy strong { font-size: 22px; font-weight: 600; letter-spacing: -.04em; }.destination-copy small { color: #7e8b82; font: 11px 'DM Mono', monospace; }.destination-card-primary .destination-copy small { color: #b4c1b5; }.destination-arrow { position: absolute; right: 27px; bottom: 25px; color: #a4b338; font-size: 21px; }.menu-footer { position: absolute; right: 6vw; bottom: 30px; left: 6vw; display: flex; justify-content: space-between; color: #8b978f; font: 9px 'DM Mono', monospace; letter-spacing: .1em; }.menu-footer b { color: #a4b338; }.menu-orbit { position: absolute; border: 1px solid #dce2d6; border-radius: 50%; pointer-events: none; }.menu-orbit-one { top: -28vw; right: -14vw; width: 60vw; height: 60vw; }.menu-orbit-two { bottom: -42vw; left: -21vw; width: 70vw; height: 70vw; border-color: #e1e6dc; }.blank-content { position: relative; z-index: 1; max-width: 1080px; margin: 0 auto; padding: 18vh 6vw; }.blank-content p { max-width: 340px; }.blank-shell .menu-header { background: #f1f3ed; }
.workspace { flex: 1; display: grid; grid-template-columns: minmax(260px, 28%) 1fr; min-height: 680px; padding: 4.5vw 4.2vw 3vw; gap: 5vw; }.sidebar { display: flex; flex-direction: column; max-width: 330px; padding: 17px 0 0; }.eyebrow, .section-label, .stage-head, .page-footer { color: #8e9892; font: 10px 'DM Mono', monospace; letter-spacing: .1em; text-transform: uppercase; }.eyebrow { color: #a7b834; }.sidebar h1 { margin: 21px 0 17px; color: #26322e; font-size: clamp(40px, 4.7vw, 68px); line-height: .98; letter-spacing: -.065em; }.sidebar h1 em { color: #a2af31; font-style: normal; }.intro { max-width: 260px; margin: 0 0 47px; color: #7b8580; font-size: 13px; line-height: 1.8; }.asset-card { display: flex; align-items: center; gap: 12px; padding: 10px; background: #fff; border: 1px solid #e3e5df; box-shadow: 0 7px 24px #313b2412; }.asset-preview { position: relative; display: grid; place-items: center; width: 48px; height: 48px; color: #fff; background: #303d38; font: 9px 'DM Mono', monospace; }.asset-file { padding: 4px 3px; border: 1px solid #89968d; }.asset-corner { position: absolute; top: 4px; right: 5px; color: #cad857; }.asset-info { display: flex; flex: 1; flex-direction: column; gap: 5px; min-width: 0; }.asset-info strong { overflow: hidden; color: #36413d; font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }.asset-info span { color: #9ca49f; font: 10px 'DM Mono', monospace; }.status-check { display: grid; place-items: center; width: 20px; height: 20px; color: #99ad2d; border: 1px solid #d8e1a1; border-radius: 50%; font: 11px 'DM Mono', monospace; }.status-check.loading { width: auto; height: auto; padding: 3px; border: 0; color: #9da69f; font-size: 9px; }.section-label { margin: 48px 0 15px; color: #afb6b0; }.details-list { margin: 0; border-top: 1px solid #dfe2dc; }.details-list div { display: flex; justify-content: space-between; padding: 11px 0; border-bottom: 1px solid #dfe2dc; }.details-list dt { color: #8b948e; font-size: 11px; }.details-list dd { margin: 0; color: #4e5c55; font: 10px 'DM Mono', monospace; }.sidebar-footer { display: flex; align-items: center; gap: 12px; margin-top: auto; color: #9da49e; font: 10px 'DM Mono', monospace; line-height: 1.7; }.footer-spark { color: #c3d34e; font-size: 24px; }
.stage-panel { min-width: 0; display: flex; flex-direction: column; }.stage-head { display: flex; justify-content: space-between; padding: 8px 0 14px; border-bottom: 1px solid #dfe2dc; }.stage-coordinate { color: #aab1ac; }.stage-coordinate b { color: #69756e; font-weight: 400; }.canvas-host { position: relative; flex: 1; min-height: 510px; overflow: hidden; background: #e7e9e4; }.canvas-host canvas { position: relative; z-index: 1; display: block; width: 100%; height: 100%; }.stage-grid { position: absolute; inset: 0; z-index: 0; opacity: .32; background-image: linear-gradient(#c6ccc6 1px, transparent 1px), linear-gradient(90deg, #c6ccc6 1px, transparent 1px); background-size: 60px 60px; mask-image: linear-gradient(to bottom, transparent, #000 40%, transparent); }.canvas-badge { position: absolute; z-index: 2; right: 24px; bottom: 23px; padding: 8px 10px; color: #a2aaa4; border: 1px solid #cbd1cc; background: #e7e9e4cc; font: 9px 'DM Mono', monospace; letter-spacing: .08em; }.canvas-badge span { margin-left: 11px; color: #9caf30; }.axis-widget { position: absolute; z-index: 2; right: 23px; top: 23px; width: 43px; height: 43px; border: 1px solid #c9d0ca; border-radius: 50%; color: #919b95; font: 8px 'DM Mono', monospace; }.axis-widget i { position: absolute; left: 21px; top: 8px; width: 1px; height: 26px; background: #b6c0b9; }.axis-widget span { position: absolute; }.axis-y { top: 3px; left: 19px; color: #b1c33e; }.axis-x { bottom: 8px; left: 31px; }.axis-z { bottom: 8px; left: 8px; }.loading-state, .error-state { position: absolute; z-index: 4; inset: 0; display: grid; place-content: center; justify-items: center; gap: 13px; color: #738078; font: 10px 'DM Mono', monospace; }.loader-ring { width: 30px; height: 30px; border: 2px solid #cad2cc; border-top-color: #aabd35; border-radius: 50%; animation: spin 1s linear infinite; }.error-state { color: #65716b; text-align: center; font-family: 'Manrope', sans-serif; }.error-state button { padding: 9px 12px; border: 1px solid #b5c16a; color: #697722; background: transparent; cursor: pointer; font-size: 11px; }.stage-toolbar { display: flex; justify-content: space-between; padding-top: 13px; }.toolbar-group { display: flex; gap: 5px; }.tool-button, .icon-button { display: flex; align-items: center; gap: 8px; border: 1px solid transparent; color: #8e9992; background: transparent; cursor: pointer; font-size: 10px; }.tool-button { padding: 7px 9px; }.tool-button.active { color: #52631d; border-color: #d6dfa7; background: #f1f5df; }.rotate-icon, .wire-icon { color: #a7b837; font-size: 18px; }.wire-icon { font-size: 16px; }.icon-button { justify-content: center; width: 32px; height: 30px; border-color: #d6dad4; font-size: 18px; }.icon-button:hover, .tool-button:hover { border-color: #b8c877; color: #596a23; }.page-footer { display: flex; justify-content: space-between; padding: 0 4.2vw 25px; font-size: 9px; }.page-footer b { color: #c4d451; padding: 0 8px; }.page-footer span:nth-child(2) { color: #b1b7b1; text-transform: none; letter-spacing: 0; }.visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }
.hover-card { position: absolute; z-index: 5; width: 190px; padding: 14px; pointer-events: none; color: #eaf0df; background: #263530ed; border: 1px solid #71806d; box-shadow: 0 12px 28px #1c282444; transform: translateZ(0); }.hover-card-kicker { margin-bottom: 7px; color: #cbd952; font: 9px 'DM Mono', monospace; letter-spacing: .1em; text-transform: uppercase; }.hover-card strong { display: block; overflow: hidden; font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }.hover-card-part { margin-top: 4px; overflow: hidden; color: #aebbb1; font: 9px 'DM Mono', monospace; text-overflow: ellipsis; white-space: nowrap; }.hover-card dl { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin: 13px 0 0; padding-top: 10px; border-top: 1px solid #52625a; }.hover-card dl div { min-width: 0; }.hover-card dt { color: #91a097; font: 8px 'DM Mono', monospace; text-transform: uppercase; }.hover-card dd { margin: 4px 0 0; color: #f2f5e9; font: 10px 'DM Mono', monospace; }
@media (max-width: 760px) { .menu-header { height: 70px; padding: 0 20px; }.menu-kicker { display: none; }.menu-content { padding: 15vh 20px 110px; }.menu-intro h1, .blank-content h1 { font-size: 54px; }.destination-grid { grid-template-columns: 1fr; gap: 12px; margin-top: 58px; }.destination-card { min-height: 210px; }.menu-footer { right: 20px; bottom: 22px; left: 20px; font-size: 8px; }.blank-content { padding: 18vh 20px; }.back-link { font-size: 9px; }.menu-orbit-one { top: -25vw; right: -45vw; width: 110vw; height: 110vw; }.menu-orbit-two { bottom: -30vw; left: -55vw; width: 130vw; height: 130vw; } }
@keyframes spin { to { transform: rotate(360deg); } } @media (max-width: 760px) { .topbar { height: 65px; padding: 0 20px; }.topbar-meta { display: none; }.upload-button { margin-left: auto; }.workspace { display: block; min-height: auto; padding: 34px 20px 42px; }.sidebar { max-width: none; padding: 0; }.sidebar h1 { margin-top: 16px; font-size: 48px; }.intro { margin-bottom: 28px; }.section-label { margin-top: 30px; }.sidebar-footer { display: none; }.stage-panel { margin-top: 42px; }.stage-coordinate { display: none; }.canvas-host { min-height: 62vh; }.page-footer { padding: 0 20px 20px; font-size: 8px; }.page-footer span:nth-child(2) { display: none; } }
</style>
