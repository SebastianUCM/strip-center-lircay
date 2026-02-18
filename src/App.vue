<script setup>
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'
import L from 'leaflet'
import logoCugat from './assets/logos/cugat.png'
import logoKfc from './assets/logos/kfc.png'
import logoPapaJohns from './assets/logos/papajohns.png'
import logoCruzVerde from './assets/logos/cruzverde.png'

// Contadores animados
const terreno = ref(0)
const estacionamientos = ref(0)
const locales = ref(0)
const countDuration = 2000

function animateValue(r, end) {
  const start = 0
  const startTime = performance.now()
  const step = (now) => {
    const elapsed = now - startTime
    const progress = Math.min(elapsed / countDuration, 1)
    const easeOut = 1 - Math.pow(1 - progress, 3)
    r.value = Math.floor(start + (end - start) * easeOut)
    if (progress < 1) requestAnimationFrame(step)
    else r.value = end
  }
  requestAnimationFrame(step)
}

const localesDisponibles = [
  { id: '07', m2: 85 },
  { id: '08', m2: 120 },
  { id: '10', m2: 95 },
  { id: '12', m2: 110 },
  { id: '13', m2: 75 },
]

const marcas = [
  { nombre: 'Supermercado Cugat', logo: logoCugat },
  { nombre: 'KFC', logo: logoKfc },
  { nombre: "Papa John's", logo: logoPapaJohns },
  { nombre: 'Cruz Verde', logo: logoCruzVerde },
]

// Polo gastronómico: KFC y Papa John's con drive-thru (imagen = foto del local en public/images/polo/)
const poloGastronomico = [
  { nombre: 'KFC', logo: logoKfc, driveThru: true, imagen: '/images/polo/kfc.jpg' },
  { nombre: "Papa John's", logo: logoPapaJohns, driveThru: true, imagen: '/images/polo/papajohns.jpg' },
]
const poloImagenError = ref({})
function setPoloImagenError(nombre) {
  poloImagenError.value[nombre] = true
}
// Solo la tarjeta sobre la que está el cursor hace el flip (independiente por card)
const poloHovered = ref(null)

const whatsappNumber = '56912345678'
const whatsappBase = `https://wa.me/${whatsappNumber}`

const videoSrc = '/videos/drone.mp4'
const videoError = ref(false)
function onVideoError() {
  videoError.value = true
}

// Modal Cotizar
const showCotizarModal = ref(false)
const formCotizar = ref({
  nombre: '',
  email: '',
  telefono: '',
  local: '',
  mensaje: '',
})

function openCotizarModal() {
  showCotizarModal.value = true
  if (typeof AOS !== 'undefined') AOS.refresh()
}
function closeCotizarModal() {
  showCotizarModal.value = false
}

function enviarCotizacion() {
  const f = formCotizar.value
  const text = encodeURIComponent(
    `Hola, me interesa cotizar un local en Strip Center Lircay.\nNombre: ${f.nombre}\nEmail: ${f.email}\nTel: ${f.telefono}\nLocal de interés: ${f.local || 'Por definir'}\nMensaje: ${f.mensaje || '-'}`
  )
  window.open(`${whatsappBase}?text=${text}`, '_blank')
  closeCotizarModal()
}

// Planta comercial: hover tooltip
const hoverLocal = ref(null)

// Marcadores del mapa con logos (coordenadas Talca)
const mapContainer = ref(null)
let mapInstance = null
const stripCenterCoords = [-35.41137356951123, -71.6515643882412]
// Oriente = este (lng menos negativo). Strip en -71.65156; hitos al este ~ -71.648 a -71.638
const mapMarkers = [
  { lat: -35.41137356951123, lng: -71.6515643882412, label: 'Strip Center Lircay', logo: null },
  { lat: -35.4112, lng: -71.6518, label: 'Supermercado Cugat', logo: logoCugat },
  { lat: -35.424, lng: -71.648, label: 'Universidad de Talca', logo: '/images/hitos/utalca.svg' },
  { lat: -35.4125, lng: -71.646, label: 'Colegios zona oriente', logo: '/images/hitos/colegio.svg' },
  { lat: -35.4105, lng: -71.644, label: 'Establecimientos educacionales', logo: '/images/hitos/colegio.svg' },
  { lat: -35.413, lng: -71.647, label: 'Edificios y equipamiento', logo: '/images/hitos/edificios.svg' },
  { lat: -35.409, lng: -71.642, label: 'Zona edificada oriente', logo: '/images/hitos/edificios.svg' },
  { lat: -35.434, lng: -71.664, label: 'Ruta 5 Sur y Acceso Norte', logo: '/images/hitos/ruta5.svg' },
  { lat: -35.421, lng: -71.647, label: 'Condominios Manuel Larraín y Valle del Maule', logo: '/images/hitos/condominios.svg' },
  { lat: -35.429, lng: -71.657, label: 'SAPU y Colegios cercanos', logo: '/images/hitos/sapu.svg' },
]

// Servicios y áreas comunes
const serviciosComunes = [
  { titulo: 'Administración', detalle: '$25,40 / m²', icono: 'admin' },
  { titulo: 'Comedor para empleados y baños públicos', detalle: 'Áreas comunes', icono: 'comedor' },
  { titulo: 'Estacionamiento para bicicletas', detalle: '16 cupos', icono: 'bici' },
  { titulo: 'Instalaciones de basura y combustible', detalle: 'Bajo normativa vigente', icono: 'instalaciones' },
]

// Bento grid: imágenes en public/images/bento/ (se excluye la foto 5)
const bentoImages = [
  '/images/bento/1.jpg',
  '/images/bento/2.jpg',
  '/images/bento/3.jpg',
  '/images/bento/4.jpg',
  '/images/bento/5.jpg',
  '/images/bento/6.jpg',
]
const bentoImagesToShow = computed(() => bentoImages.filter((_, i) => i !== 4))
const bentoImageError = ref({})
function markBentoError(idx) {
  bentoImageError.value[idx] = true
}

// Planta comercial: imagen real con logos (public/images/planta/)
const plantaConLogosSrc = '/images/planta/planta-con-logos.PNG'

// Vista drone: imagen + puntos que identifican cada local (public/images/planta/)
const droneVistaSrc = '/images/planta/drone-vista.jpeg'
const dronePuntos = [
  { localId: '07', m2: 85, left: '18%', top: '28%' },
  { localId: '08', m2: 120, left: '26%', top: '28%' },
  { localId: '10', m2: 95, left: '48%', top: '38%' },
  { localId: '12', m2: 110, left: '58%', top: '38%' },
  { localId: '13', m2: 75, left: '66%', top: '55%' },
]
const dronePuntoActivo = ref(null)

// Accesos desde drone: Avenida Lircay y 14 Norte (public/images/planta/)
const accesoLircaySrc = '/images/planta/acceso-lircay.jpeg'
const acceso14NorteSrc = '/images/planta/acceso-14-norte.jpeg'
const accesoLircayError = ref(false)
const acceso14NorteError = ref(false)

// Nav sticky: añadir sombra al hacer scroll
const navScrolled = ref(false)
function onScroll() {
  navScrolled.value = window.scrollY > 50
}

// Posiciones de cada local sobre planta-con-logos.PNG (% del ancho/alto de la imagen)
// Ajustar left/top/width/height si las zonas no coinciden con los recuadros en la planta
function getLocalZoneStyle(localId) {
  const zones = {
    // Locales al norte de Cugat (primeros recuadros pequeños del edificio principal)
    '07': { left: '7%', top: '18%', width: '7.5%', height: '16%' },
    '08': { left: '15%', top: '18%', width: '7.5%', height: '16%' },
    // Entre Cruz Verde y KFC (fila de locales con numeración)
    '10': { left: '43%', top: '32%', width: '8%', height: '16%' },
    '12': { left: '54%', top: '32%', width: '8%', height: '16%' },
    '13': { left: '65%', top: '54%', width: '7.5%', height: '16%' },
  }
  const z = zones[localId]
  if (!z) return {}
  return {
    left: z.left,
    top: z.top,
    width: z.width,
    height: z.height,
  }
}

onMounted(() => {
  animateValue(terreno, 11800)
  animateValue(estacionamientos, 143)
  animateValue(locales, 15)
  window.addEventListener('scroll', onScroll, { passive: true })
  if (typeof AOS !== 'undefined') {
    AOS.init({ duration: 700, once: true, offset: 40 })
  }
  nextTick(() => {
    if (!mapContainer.value) return
    mapInstance = L.map(mapContainer.value).setView(stripCenterCoords, 15)
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    }).addTo(mapInstance)
    const stripCenterIcon = L.divIcon({
      className: 'strip-center-marker',
      html: '<span style="background:#f37021;color:white;width:44px;height:44px;border-radius:50%;border:4px solid #1e3a8a;display:flex;align-items:center;justify-content:center;font-size:22px;box-shadow:0 4px 12px rgba(0,0,0,0.4);font-weight:bold">🛒</span>',
      iconSize: [44, 44],
      iconAnchor: [22, 22],
    })
    function createLogoIcon(logoUrl, label) {
      const escapedLabel = label.replace(/'/g, "\\'")
      return L.divIcon({
        className: 'hito-logo-marker',
        html: `<div style="width:42px;height:42px;border-radius:10px;overflow:hidden;border:3px solid white;box-shadow:0 2px 10px rgba(0,0,0,0.35);background:#fff;display:flex;align-items:center;justify-content:center"><img src="${logoUrl}" alt="${escapedLabel}" style="width:100%;height:100%;object-fit:contain;padding:3px" /></div>`,
        iconSize: [42, 42],
        iconAnchor: [21, 21],
      })
    }
    const defaultIcon = L.divIcon({
      className: 'custom-marker',
      html: '<span style="background:#f37021;color:white;width:32px;height:32px;border-radius:50%;border:3px solid white;display:flex;align-items:center;justify-content:center;font-size:16px;box-shadow:0 2px 6px rgba(0,0,0,0.3)">📍</span>',
      iconSize: [32, 32],
      iconAnchor: [16, 16],
    })
    L.circle(stripCenterCoords, { radius: 120, color: '#f37021', fillColor: '#f37021', fillOpacity: 0.15, weight: 2 }).addTo(mapInstance)
    mapMarkers.forEach((m) => {
      const isStripCenter = m.label === 'Strip Center Lircay'
      const icon = isStripCenter ? stripCenterIcon : (m.logo ? createLogoIcon(m.logo, m.label) : defaultIcon)
      const marker = L.marker([m.lat, m.lng], { icon })
        .addTo(mapInstance)
        .bindPopup(`<strong>${m.label}</strong>`)
      if (isStripCenter) marker.bindTooltip('Strip Center Lircay', { permanent: true, direction: 'top', className: 'strip-center-tooltip' })
    })
  })
})
onBeforeUnmount(() => {
  window.removeEventListener('scroll', onScroll)
})
</script>

<template>
  <div class="min-h-screen text-[#1e3a8a] relative overflow-x-hidden">
    <!-- Fondo elegante: textura en transparencia + gradientes suaves (no afecta interacción) -->
    <div class="page-bg-elegant" aria-hidden="true"></div>
    <div class="relative z-10">
    <!-- Nav sticky -->
    <nav
      class="fixed top-0 left-0 right-0 z-40 transition-all duration-300 ease-out"
      :class="navScrolled ? 'bg-white/98 backdrop-blur-md shadow-lg py-3' : 'bg-transparent py-5'"
      role="navigation"
      aria-label="Principal"
    >
      <div class="max-w-6xl mx-auto px-4 sm:px-6 flex items-center justify-between">
        <a href="#" class="font-heading font-bold text-lg md:text-xl tracking-tight text-white hover:text-white/90 transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-white focus-visible:ring-offset-2 focus-visible:ring-offset-transparent rounded" :class="navScrolled ? 'text-[#1e3a8a] hover:text-[#1e3a8a]/90 focus-visible:ring-[#1e3a8a]' : ''">
          Strip Center Lircay
        </a>
        <div class="hidden md:flex items-center gap-8">
          <a href="#ubicacion" class="text-sm font-medium transition-colors rounded focus:outline-none focus-visible:ring-2 focus-visible:ring-[#f37021] focus-visible:ring-offset-2" :class="navScrolled ? 'text-gray-600 hover:text-[#1e3a8a]' : 'text-white/90 hover:text-white'">Ubicación</a>
          <a href="#locales" class="text-sm font-medium transition-colors rounded focus:outline-none focus-visible:ring-2 focus-visible:ring-[#f37021] focus-visible:ring-offset-2" :class="navScrolled ? 'text-gray-600 hover:text-[#1e3a8a]' : 'text-white/90 hover:text-white'">Locales</a>
          <button type="button" class="btn-primary text-sm py-2.5 px-5 rounded-xl" @click="openCotizarModal" aria-label="Abrir formulario de cotización">Cotizar</button>
        </div>
      </div>
    </nav>

    <!-- Hero -->
    <header class="relative h-[90vh] min-h-[560px] flex items-center justify-center overflow-hidden">
      <div class="absolute inset-0 bg-gradient-to-br from-[#1e3a8a] via-[#1e3a8a] to-[#0f172a]" />
      <video
        v-show="!videoError"
        autoplay
        muted
        loop
        playsinline
        class="absolute inset-0 w-full h-full object-cover"
        :src="videoSrc"
        @error="onVideoError"
      ></video>
      <div class="absolute inset-0 bg-black/40" />
      <div class="relative z-10 px-4 text-center text-white max-w-4xl mx-auto pt-12">
        <span class="inline-block px-4 py-1.5 rounded-full bg-white/10 text-[#f37021] font-semibold tracking-widest uppercase text-xs md:text-sm mb-4" data-aos="fade-down">
          Talca · Región del Maule
        </span>
        <h1 class="font-heading text-3xl sm:text-4xl md:text-5xl lg:text-6xl font-bold leading-[1.15] tracking-tight" style="letter-spacing: -0.03em" data-aos="fade-up" data-aos-delay="100">
          Oportunidad de Inversión en el Corazón Comercial de Talca
        </h1>
        <p class="mt-5 text-lg md:text-xl text-white/90 max-w-2xl mx-auto leading-relaxed" data-aos="fade-up" data-aos-delay="200">
          Ubicación privilegiada. Marcas ancla consolidadas. Alto flujo vehicular.
        </p>
        <a
          href="#locales"
          class="btn-primary mt-8 px-8 py-4 text-base shadow-xl hover:shadow-2xl hover:scale-[1.02] rounded-xl"
          data-aos="fade-up"
          data-aos-delay="300"
          aria-label="Ir a sección de locales disponibles"
        >
          Ver locales disponibles
        </a>
      </div>
    </header>

    <!-- Cifras clave -->
    <section class="py-20 md:py-28 bg-[#1e3a8a] text-white section-cifras-bg">
      <div class="max-w-6xl mx-auto px-4">
        <h2 class="section-title text-white/95 mb-2" data-aos="fade-up">
          Cifras que respaldan tu inversión
        </h2>
        <span class="section-title-accent bg-white/30" data-aos="fade-up"></span>
        <p class="section-subtitle text-white/80 mb-12 mt-6" data-aos="fade-up" data-aos-delay="50">
          Proyecto consolidado con números que hablan por sí solos.
        </p>
        <div class="grid grid-cols-1 md:grid-cols-3 gap-10 md:gap-14">
          <div class="text-center transition-transform duration-200 hover:scale-[1.03]" data-aos="fade-up" data-aos-delay="0">
            <p class="font-heading text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">{{ terreno.toLocaleString('es-CL') }}</p>
            <p class="text-base md:text-lg mt-2 text-white/90">m² de terreno</p>
          </div>
          <div class="text-center transition-transform duration-200 hover:scale-[1.03]" data-aos="fade-up" data-aos-delay="100">
            <p class="font-heading text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">{{ estacionamientos }}</p>
            <p class="text-base md:text-lg mt-2 text-white/90">estacionamientos</p>
          </div>
          <div class="text-center transition-transform duration-200 hover:scale-[1.03]" data-aos="fade-up" data-aos-delay="200">
            <p class="font-heading text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">{{ locales }}</p>
            <p class="text-base md:text-lg mt-2 text-white/90">locales totales</p>
          </div>
        </div>
      </div>
    </section>

    <!-- Polo Gastronómico Consolidado -->
    <section class="py-20 md:py-28 bg-white/95 backdrop-blur-[2px]">
      <div class="max-w-6xl mx-auto px-4">
        <h2 class="section-title mb-2" data-aos="fade-up">
          Polo Gastronómico Consolidado
        </h2>
        <span class="section-title-accent" data-aos="fade-up"></span>
        <p class="section-subtitle mb-12 mt-6" data-aos="fade-up" data-aos-delay="100">
          Marcas líderes que generan tráfico constante y valor para tu inversión.
        </p>
        <div class="grid md:grid-cols-2 gap-8">
          <div
            v-for="(polo, i) in poloGastronomico"
            :key="polo.nombre"
            class="group relative rounded-2xl overflow-hidden card-premium hover:-translate-y-1"
            :class="{ 'polo-flip-active': poloHovered === polo.nombre }"
            data-aos="fade-up"
            :data-aos-delay="150 + i * 100"
            @mouseenter="poloHovered = polo.nombre"
            @mouseleave="poloHovered = null"
          >
            <!-- Flip: al pasar el cursor el logo gira y se muestra la imagen del local -->
            <div class="aspect-[4/3] overflow-hidden rounded-t-2xl polo-flip-wrap">
              <div class="relative w-full h-full polo-flip-inner">
                <!-- Cara frontal: logo (gira 180° y se oculta al hover) -->
                <div class="polo-flip-front absolute inset-0 flex items-center justify-center p-8 bg-gradient-to-br from-gray-100 to-gray-200">
                  <img
                    :src="polo.logo"
                    :alt="polo.nombre"
                    class="polo-logo-img max-h-24 md:max-h-32 w-auto object-contain"
                  />
                </div>
                <!-- Cara trasera: imagen del local (visible al hover). Inner con translateZ evita que la imagen desaparezca al girar. -->
                <div class="polo-flip-back absolute inset-0 bg-gray-100">
                  <div class="polo-flip-back-inner absolute inset-0">
                    <img
                      v-show="!poloImagenError[polo.nombre]"
                      :src="polo.imagen"
                      :alt="polo.nombre + ' — vista del local'"
                      class="absolute inset-0 w-full h-full object-cover"
                      @error="setPoloImagenError(polo.nombre)"
                    />
                    <div
                      v-show="poloImagenError[polo.nombre]"
                      class="absolute inset-0 flex flex-col items-center justify-center text-gray-500 text-sm p-4 text-center"
                    >
                      <span>Coloque <strong>{{ polo.nombre === 'KFC' ? 'kfc.jpg' : 'papajohns.jpg' }}</strong> en <code class="bg-gray-300 px-1 rounded text-xs">public/images/polo/</code></span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="p-6 text-center">
              <h3 class="font-heading text-xl font-bold text-[#1e3a8a]">{{ polo.nombre }}</h3>
              <span
                v-if="polo.driveThru"
                class="inline-flex items-center gap-2 mt-3 px-4 py-2 rounded-full bg-[#f37021]/10 text-[#f37021] font-semibold text-sm"
              >
                <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20"><path d="M8 16.5a1.5 1.5 0 11-3 0 1.5 1.5 0 013 0zM15 16.5a1.5 1.5 0 11-3 0 1.5 1.5 0 013 0z" /><path d="M3 4a1 1 0 00-1 1v10a1 1 0 001 1h1.05a2.5 2.5 0 014.9 0H10a1 1 0 001-1V5a1 1 0 00-1-1H3zM14 7a1 1 0 00-1 1v6.05A2.5 2.5 0 0115.95 16H17a1 1 0 001-1v-5a1 1 0 00-.293-.707l-2-2A1 1 0 0015 7h-1z" /></svg>
                Servicio Drive-Thru disponible
              </span>
            </div>
          </div>
        </div>
      </div>
    </section>

    <!-- Marcas ancla - slider -->
    <section class="py-16 md:py-20 bg-gray-50/90 border-y border-gray-100/80">
      <h2 class="section-title text-xl md:text-2xl mb-2 px-4" data-aos="fade-up">Marcas ancla</h2>
      <span class="section-title-accent block" data-aos="fade-up"></span>
      <div class="overflow-hidden mt-10">
        <div class="slider-track gap-12 md:gap-20 px-4">
          <template v-for="(_, i) in 2" :key="'set-' + i">
            <div
              v-for="m in marcas"
              :key="m.nombre + i"
              class="flex-shrink-0 flex items-center justify-center w-48 md:w-56 h-20 md:h-24 bg-white rounded-xl shadow-sm border border-gray-100 px-6 transition-all duration-300 hover:scale-105 hover:shadow-md"
            >
              <img v-if="m.logo" :src="m.logo" :alt="m.nombre" class="max-h-14 md:max-h-16 w-auto object-contain block" />
              <span v-else class="font-semibold text-[#1e3a8a] text-sm">{{ m.nombre }}</span>
            </div>
          </template>
        </div>
      </div>
    </section>

    <!-- Infraestructura: 143 estacionamientos + Bento grid -->
    <section class="py-20 md:py-28 bg-white/95 backdrop-blur-[2px]">
      <div class="max-w-6xl mx-auto px-4">
        <h2 class="section-title mb-2" data-aos="fade-up">
          Infraestructura de Alto Tráfico
        </h2>
        <span class="section-title-accent" data-aos="fade-up"></span>
        <p class="section-subtitle mb-4 mt-6" data-aos="fade-up" data-aos-delay="100">
          Facilidad de acceso y comodidad garantizada con <strong class="text-[#1e3a8a]">143 calzos estratégicos</strong>.
        </p>
        <div class="grid grid-cols-2 md:grid-cols-4 gap-3 md:gap-4 mt-12">
          <div
            v-for="(img, idx) in bentoImagesToShow"
            :key="idx"
            class="relative aspect-square md:aspect-[4/3] rounded-xl overflow-hidden bg-gradient-to-br from-[#1e3a8a]/10 to-[#f37021]/10 border border-gray-200 flex items-center justify-center shadow-md hover:shadow-xl transition-all duration-300 hover:scale-[1.02]"
            :class="idx === 0 ? 'col-span-2 md:col-span-2 md:row-span-2 md:aspect-auto' : ''"
            data-aos="zoom-in"
            :data-aos-delay="50 * idx"
          >
            <img
              :src="img"
              :alt="'Estacionamiento ' + (idx + 1)"
              class="w-full h-full object-cover"
              @error="markBentoError(idx)"
            />
            <span
              v-show="bentoImageError[idx] === true"
              class="absolute inset-0 flex items-center justify-center text-[#1e3a8a]/40 text-4xl font-bold bg-gradient-to-br from-[#1e3a8a]/10 to-[#f37021]/10"
            >
              {{ idx + 1 }}
            </span>
          </div>
        </div>
      </div>
    </section>

    <!-- Mapa de contexto urbano interactivo -->
    <section id="ubicacion" class="py-20 md:py-28 bg-gray-50/90">
      <div class="max-w-5xl mx-auto px-4">
        <h2 class="section-title mb-2" data-aos="fade-up">
          Ubicación Estratégica
        </h2>
        <span class="section-title-accent" data-aos="fade-up"></span>
        <p class="section-subtitle mb-10 mt-6" data-aos="fade-up" data-aos-delay="100">
          Conectividad con los principales ejes y servicios de Talca.
        </p>
        <div class="relative rounded-2xl overflow-hidden bg-gray-200 aspect-[16/10] min-h-[300px] border border-gray-200 shadow-lg" data-aos="fade-up">
          <div ref="mapContainer" class="absolute inset-0 w-full h-full rounded-2xl z-0" style="min-height: 300px;"></div>
        </div>
        <p class="text-center text-gray-600 mt-6 text-sm">
          Universidad de Talca · Ruta 5 Sur · Condominios Manuel Larraín y Valle del Maule · SAPU y colegios cercanos
        </p>
      </div>
    </section>

    <!-- Planta comercial interactiva + Servicios y áreas comunes -->
    <section class="py-20 md:py-28 bg-white/95 backdrop-blur-[2px]">
      <div class="max-w-6xl mx-auto px-4">
        <h2 class="section-title mb-2" data-aos="fade-up">
          Planta Comercial y Áreas Comunes
        </h2>
        <span class="section-title-accent" data-aos="fade-up"></span>
        <p class="section-subtitle mb-10 mt-6" data-aos="fade-up" data-aos-delay="100">
          Locales disponibles con superficies optimizadas. Pasa el cursor sobre cada local para ver m².
        </p>

        <!-- Planta comercial real con logos (Cugat, Cruz Verde, KFC, Papa John's) -->
        <div class="relative max-w-4xl mx-auto mb-14 rounded-2xl overflow-hidden border border-gray-200 shadow-lg" data-aos="fade-up">
          <img
            :src="plantaConLogosSrc"
            alt="Planta comercial Strip Center Lircay — Cugat, Cruz Verde, KFC, Papa John's y locales disponibles"
            class="w-full h-auto block"
          />
          <!-- Zonas verdes: locales disponibles (07, 08, 10, 12, 13) -->
          <div
            v-for="loc in localesDisponibles"
            :key="'zone-' + loc.id"
            class="absolute rounded border-2 border-green-500 bg-green-500/25 hover:bg-green-500/40 hover:border-green-600 transition-all duration-200 cursor-pointer z-10"
            :style="getLocalZoneStyle(loc.id)"
            @mouseenter="hoverLocal = loc.id"
            @mouseleave="hoverLocal = null"
          >
            <div
              v-if="hoverLocal === loc.id"
              class="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-3 py-2 bg-[#1e3a8a] text-white text-sm font-semibold rounded-lg shadow-xl whitespace-nowrap z-20"
            >
              Local {{ loc.id }} — {{ loc.m2 }} m² · Cotizar
            </div>
          </div>
        </div>
        <p class="text-center text-gray-500 text-sm mb-2">
          <span class="inline-block w-3 h-3 rounded-sm bg-green-500/40 border border-green-600 align-middle mr-1"></span>
          En verde: locales disponibles. Pasa el cursor para ver m².
        </p>

        <!-- Vista desde drone: bloque con espacio generoso debajo -->
        <div class="mt-20 mb-24 md:mt-24 md:mb-28" data-aos="fade-up">
          <div class="bg-gradient-to-b from-gray-50/80 to-white rounded-3xl p-6 md:p-8 border border-gray-100 shadow-inner">
            <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-3 mb-6">
              <div>
                <span class="inline-flex items-center gap-2 text-[#f37021] font-semibold text-sm uppercase tracking-wider mb-1">
                  <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3.055 11H5a2 2 0 012 2v1a2 2 0 002 2 2 2 0 012 2v2.945M8 3.935V5.5A2.5 2.5 0 0010.5 8h.5a2 2 0 012 2 2 2 0 104 0h.5a2.5 2.5 0 0010.5-4V3.935M21 12a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
                  Vista aérea
                </span>
                <h3 class="font-heading text-xl md:text-2xl font-bold text-[#1e3a8a] tracking-tight">
                  Vista desde drone
                </h3>
              </div>
              <p class="text-gray-600 text-sm max-w-xs">
                Pasa el cursor sobre cada punto para identificar el local.
              </p>
            </div>
            <div class="relative rounded-2xl overflow-hidden border border-gray-200 shadow-lg bg-white">
              <img
                :src="droneVistaSrc"
                alt="Vista aérea Strip Center Lircay — identificación de locales"
                class="w-full h-auto block bg-gray-100"
                @error="($event.target).style.display='none'; $event.target.nextElementSibling?.classList.remove('hidden')"
              />
              <div class="hidden absolute inset-0 flex items-center justify-center bg-gray-100 text-gray-500 rounded-2xl">
                Coloque <strong>drone-vista.png</strong> o <strong>drone-vista.jpg</strong> en <code class="text-sm bg-gray-200 px-1 rounded">public/images/planta/</code>
              </div>
              <template v-for="p in dronePuntos" :key="'drone-' + p.localId">
                <button
                  type="button"
                  class="absolute w-7 h-7 rounded-full bg-[#f37021] border-2 border-white shadow-lg cursor-pointer z-10 transform -translate-x-1/2 -translate-y-1/2 hover:scale-125 focus:scale-125 focus:outline-none focus-visible:ring-2 focus-visible:ring-[#f37021] focus-visible:ring-offset-2 transition-transform"
                  :style="{ left: p.left, top: p.top }"
                  @mouseenter="dronePuntoActivo = p.localId"
                  @mouseleave="dronePuntoActivo = null"
                  @focus="dronePuntoActivo = p.localId"
                  @blur="dronePuntoActivo = null"
                >
                  <span class="sr-only">Local {{ p.localId }}</span>
                </button>
                <div
                  v-show="dronePuntoActivo === p.localId"
                  class="absolute z-20 px-3 py-2 bg-[#1e3a8a] text-white text-sm font-semibold rounded-lg shadow-xl whitespace-nowrap pointer-events-none"
                  :style="{ left: p.left, top: p.top, transform: 'translate(-50%, calc(-100% - 12px))' }"
                >
                  Local {{ p.localId }} — {{ p.m2 }} m²
                </div>
              </template>
            </div>

            <!-- Accesos: Avenida Lircay y 14 Norte -->
            <p class="font-heading text-base font-semibold text-[#1e3a8a] mt-8 mb-4" data-aos="fade-up">Accesos al strip</p>
            <div class="grid md:grid-cols-2 gap-6">
              <div
                class="group acceso-card rounded-2xl overflow-hidden border border-gray-200 shadow-md bg-gray-50 transition-all duration-300 hover:shadow-xl hover:border-[#f37021]/30 hover:-translate-y-1"
                data-aos="fade-up"
                data-aos-delay="100"
              >
                <div class="relative aspect-video bg-gray-200 overflow-hidden">
                  <img
                    v-show="!accesoLircayError"
                    :src="accesoLircaySrc"
                    alt="Entrada y salida por Avenida Lircay — vista desde drone"
                    class="w-full h-full object-cover transition-transform duration-500 ease-out group-hover:scale-105"
                    @error="accesoLircayError = true"
                  />
                  <div v-show="accesoLircayError" class="absolute inset-0 flex flex-col items-center justify-center text-gray-500 p-4 text-center">
                    <svg class="w-12 h-12 text-gray-400 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" /></svg>
                    <span class="text-sm">Coloque <strong>acceso-lircay.jpeg</strong> en <code class="bg-gray-200 px-1 rounded text-xs">public/images/planta/</code></span>
                  </div>
                </div>
                <div class="p-4 bg-white border-t border-gray-100 transition-colors duration-300 group-hover:bg-[#fefcfb]">
                  <span class="inline-flex items-center gap-2 text-[#f37021] font-semibold text-sm transition-transform duration-300 group-hover:translate-x-0.5">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 20l-5.446-2.724A1 1 0 013 16.382V5.618a1 1 0 011.554-.832L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7" /></svg>
                    Entrada y salida por Avenida Lircay
                  </span>
                </div>
              </div>
              <div
                class="group acceso-card rounded-2xl overflow-hidden border border-gray-200 shadow-md bg-gray-50 transition-all duration-300 hover:shadow-xl hover:border-[#f37021]/30 hover:-translate-y-1"
                data-aos="fade-up"
                data-aos-delay="200"
              >
                <div class="relative aspect-video bg-gray-200 overflow-hidden">
                  <img
                    v-show="!acceso14NorteError"
                    :src="acceso14NorteSrc"
                    alt="Entrada y salida por 14 Norte — vista desde drone"
                    class="w-full h-full object-cover transition-transform duration-500 ease-out group-hover:scale-105"
                    @error="acceso14NorteError = true"
                  />
                  <div v-show="acceso14NorteError" class="absolute inset-0 flex flex-col items-center justify-center text-gray-500 p-4 text-center">
                    <svg class="w-12 h-12 text-gray-400 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" /></svg>
                    <span class="text-sm">Coloque <strong>acceso-14-norte.jpeg</strong> en <code class="bg-gray-200 px-1 rounded text-xs">public/images/planta/</code></span>
                  </div>
                </div>
                <div class="p-4 bg-white border-t border-gray-100 transition-colors duration-300 group-hover:bg-[#fefcfb]">
                  <span class="inline-flex items-center gap-2 text-[#f37021] font-semibold text-sm transition-transform duration-300 group-hover:translate-x-0.5">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 20l-5.446-2.724A1 1 0 013 16.382V5.618a1 1 0 011.554-.832L9 7m0 13l6-3m-6 3V7m6 10l4.553 2.276A1 1 0 0021 18.382V7.618a1 1 0 00-.553-.894L15 4m0 13V4m0 0L9 7" /></svg>
                    Entrada y salida por 14 Norte
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Servicios y áreas comunes - bloque destacado -->
        <div class="pt-6 mt-14 sm:mt-16">
          <div class="rounded-3xl overflow-hidden border border-gray-200/80 bg-gradient-to-br from-white via-[#fefcfb] to-[#f8f7fc] shadow-lg shadow-gray-200/40">
            <div class="p-6 sm:p-8 md:p-10">
              <div class="flex flex-col sm:flex-row sm:items-end sm:justify-between gap-4 mb-8">
                <div>
                  <h3 class="font-heading text-xl md:text-2xl font-bold text-[#1e3a8a] tracking-tight" data-aos="fade-up">
                    Servicios y Áreas Comunes
                  </h3>
                  <span class="section-title-accent block mt-2" data-aos="fade-up"></span>
                  <p class="text-gray-600 mt-4 max-w-xl text-sm md:text-base leading-relaxed" data-aos="fade-up" data-aos-delay="50">
                    Todo lo que necesitas para operar con comodidad: administración, áreas de uso común y servicios bajo normativa.
                  </p>
                </div>
              </div>
              <div class="grid sm:grid-cols-2 lg:grid-cols-4 gap-4 md:gap-5">
                <div
                  v-for="(s, i) in serviciosComunes"
                  :key="s.titulo"
                  class="servicio-card group relative p-5 md:p-6 rounded-2xl bg-white/80 backdrop-blur-sm border border-gray-100 hover:border-[#f37021]/30 hover:shadow-lg hover:shadow-[#f37021]/10 transition-all duration-300 overflow-hidden"
                  data-aos="fade-up"
                  :data-aos-delay="80 + i * 60"
                >
                  <div class="absolute top-0 right-0 w-24 h-24 rounded-full bg-[#f37021]/5 -translate-y-1/2 translate-x-1/2 group-hover:scale-150 group-hover:bg-[#f37021]/10 transition-all duration-500" aria-hidden="true"></div>
                  <div class="relative">
                    <div class="w-14 h-14 rounded-2xl bg-gradient-to-br from-[#f37021]/15 to-[#1e3a8a]/10 flex items-center justify-center mb-4 group-hover:scale-110 transition-transform duration-300 text-[#f37021]">
                      <!-- Admin -->
                      <svg v-if="s.icono === 'admin'" class="w-7 h-7" fill="none" stroke="currentColor" stroke-width="1.8" viewBox="0 0 24 24" aria-hidden="true"><path stroke-linecap="round" stroke-linejoin="round" d="M19 21V5a2 2 0 00-2-2H7a2 2 0 00-2 2v16m14 0h2m-2 0h-5m-9 0H3m2 0h5M9 7h1m-1 4h1m4-4h1m-1 4h1m-5 10v-5a1 1 0 011-1h2a1 1 0 011 1v5m-4 0h4" /></svg>
                      <!-- Comedor -->
                      <svg v-else-if="s.icono === 'comedor'" class="w-7 h-7" fill="none" stroke="currentColor" stroke-width="1.8" viewBox="0 0 24 24" aria-hidden="true"><path stroke-linecap="round" stroke-linejoin="round" d="M2 21h20M5 21V7l7-4 7 4v14M9 21V9l6-4 6 4v12M3 10h4v8H3v-8zm14 0h4v8h-4v-8z" /></svg>
                      <!-- Bici -->
                      <svg v-else-if="s.icono === 'bici'" class="w-7 h-7" fill="none" stroke="currentColor" stroke-width="1.8" viewBox="0 0 24 24" aria-hidden="true"><path stroke-linecap="round" stroke-linejoin="round" d="M5 12h14M5 12a2 2 0 01-2-2 2 2 0 012-2h2a2 2 0 012 2v4a2 2 0 01-2 2H7a2 2 0 01-2-2V8a2 2 0 012-2h4a2 2 0 012 2v4a2 2 0 01-2 2h-2a2 2 0 01-2-2z" /></svg>
                      <!-- Instalaciones -->
                      <svg v-else class="w-7 h-7" fill="none" stroke="currentColor" stroke-width="1.8" viewBox="0 0 24 24" aria-hidden="true"><path stroke-linecap="round" stroke-linejoin="round" d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4" /></svg>
                    </div>
                    <h4 class="font-heading font-bold text-[#1e3a8a] text-base md:text-lg leading-snug">{{ s.titulo }}</h4>
                    <p class="text-sm text-gray-600 mt-2 leading-relaxed">{{ s.detalle }}</p>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </section>

    <!-- Locales disponibles -->
    <section id="locales" class="py-20 md:py-28 bg-gray-50/90">
      <div class="max-w-4xl mx-auto px-4">
        <h2 class="section-title mb-2" data-aos="fade-up">
          Locales disponibles
        </h2>
        <span class="section-title-accent" data-aos="fade-up"></span>
        <p class="section-subtitle mb-10 mt-6" data-aos="fade-up" data-aos-delay="100">
          Consulta disponibilidad y condiciones. Cotiza sin compromiso.
        </p>
        <ul class="space-y-4">
          <li
            v-for="(loc, index) in localesDisponibles"
            :key="loc.id"
            class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 p-5 rounded-2xl card-premium hover:border-[#f37021]/20"
            data-aos="fade-up"
            :data-aos-delay="80 * index"
          >
            <div class="flex items-center gap-4">
              <span class="font-heading text-xl md:text-2xl font-bold text-[#1e3a8a]">Local {{ loc.id }}</span>
              <span class="text-gray-600 font-medium">{{ loc.m2 }} m²</span>
            </div>
            <a
              :href="`${whatsappBase}?text=${encodeURIComponent('Hola, me interesa cotizar el Local ' + loc.id + ' (' + loc.m2 + ' m²) en Strip Center Lircay.')}`"
              target="_blank"
              rel="noopener"
              class="btn-primary shrink-0"
              :aria-label="'Cotizar Local ' + loc.id"
            >
              Cotizar
            </a>
          </li>
        </ul>
      </div>
    </section>

    <!-- Cierre ubicación -->
    <section class="py-20 md:py-28 bg-[#1e3a8a] text-white section-cifras-bg">
      <div class="max-w-4xl mx-auto px-4 text-center">
        <h2 class="section-title text-white mb-2" data-aos="fade-up">
          Conectividad directa
        </h2>
        <span class="section-title-accent bg-white/40" data-aos="fade-up"></span>
        <p class="text-lg md:text-xl text-white/90 leading-relaxed mt-6" data-aos="fade-up" data-aos-delay="100">
          <strong class="text-[#f37021]">Av. Lircay</strong> y <strong class="text-[#f37021]">Ruta 5 Sur</strong>. Alto tráfico vehicular y visibilidad excepcional en el corazón comercial de Talca.
        </p>
        <p class="mt-5 text-white/75 text-sm">Strip Center Lircay — Talca, Región del Maule</p>
      </div>
    </section>

    <!-- Footer -->
    <footer class="border-t border-gray-800 bg-gray-900 py-12 text-white/80">
      <div class="max-w-6xl mx-auto px-4 sm:px-6 flex flex-col md:flex-row items-center justify-between gap-6">
        <p class="font-heading text-sm font-semibold text-white/95">Strip Center Lircay</p>
        <p class="text-sm text-white/70 text-center md:text-left">Oportunidad de inversión comercial en Talca.</p>
        <a :href="`${whatsappBase}?text=${encodeURIComponent('Hola, me interesa información sobre Strip Center Lircay.')}`" target="_blank" rel="noopener" class="btn-primary text-sm py-2.5 px-5 shrink-0">
          Contactar por WhatsApp
        </a>
      </div>
    </footer>

    <!-- CTA Flotante -->
    <button
      type="button"
      class="fixed bottom-6 right-6 z-50 px-5 py-4 bg-[#f37021] text-white font-bold rounded-full shadow-lg hover:bg-[#e06518] hover:shadow-xl hover:scale-105 active:scale-100 transition-all duration-200 text-sm md:text-base focus:outline-none focus-visible:ring-2 focus-visible:ring-[#f37021] focus-visible:ring-offset-2"
      style="margin-bottom: env(safe-area-inset-bottom, 0); margin-right: env(safe-area-inset-right, 0);"
      @click="openCotizarModal"
      aria-label="Cotizar local ahora"
    >
      Cotizar Local Ahora
    </button>

    <!-- Modal formulario Cotizar -->
    <Teleport to="body">
      <Transition name="modal">
        <div
          v-if="showCotizarModal"
          class="fixed inset-0 z-[100] flex items-center justify-center p-4 bg-black/55 backdrop-blur-sm"
          role="dialog"
          aria-modal="true"
          aria-labelledby="modal-cotizar-title"
          @click.self="closeCotizarModal"
        >
          <div class="bg-white rounded-3xl shadow-2xl max-w-md w-full p-6 md:p-8 border border-gray-100" @click.stop>
            <h3 id="modal-cotizar-title" class="font-heading text-xl font-bold text-[#1e3a8a] mb-6">Solicitar cotización</h3>
            <form @submit.prevent="enviarCotizacion" class="space-y-4">
              <input
                v-model="formCotizar.nombre"
                type="text"
                placeholder="Nombre"
                class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-[#f37021] focus:border-[#f37021] outline-none transition-shadow"
                required
              />
              <input
                v-model="formCotizar.email"
                type="email"
                placeholder="Email"
                class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-[#f37021] focus:border-[#f37021] outline-none transition-shadow"
                required
              />
              <input
                v-model="formCotizar.telefono"
                type="tel"
                placeholder="Teléfono"
                class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-[#f37021] focus:border-[#f37021] outline-none transition-shadow"
              />
              <select
                v-model="formCotizar.local"
                class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-[#f37021] focus:border-[#f37021] outline-none bg-white transition-shadow"
              >
                <option value="">Local de interés</option>
                <option v-for="loc in localesDisponibles" :key="loc.id" :value="'Local ' + loc.id + ' (' + loc.m2 + ' m²)'">
                  Local {{ loc.id }} ({{ loc.m2 }} m²)
                </option>
              </select>
              <textarea
                v-model="formCotizar.mensaje"
                placeholder="Mensaje (opcional)"
                rows="3"
                class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-[#f37021] focus:border-[#f37021] outline-none resize-none transition-shadow"
              ></textarea>
              <div class="flex gap-3 pt-2">
                <button
                  type="button"
                  class="flex-1 py-3 rounded-xl border border-gray-200 text-gray-700 font-semibold hover:bg-gray-50 focus:outline-none focus-visible:ring-2 focus-visible:ring-gray-400"
                  @click="closeCotizarModal"
                >
                  Cerrar
                </button>
                <button
                  type="submit"
                  class="flex-1 py-3 rounded-xl btn-primary"
                >
                  Enviar por WhatsApp
                </button>
              </div>
            </form>
          </div>
        </div>
      </Transition>
    </Teleport>
    </div>
  </div>
</template>

<style scoped>
.slider-track {
  display: flex;
  width: max-content;
  animation: slide 25s linear infinite;
}
@keyframes slide {
  0% { transform: translateX(0); }
  100% { transform: translateX(-50%); }
}
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.25s ease;
}
.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
.modal-enter-active .bg-white,
.modal-leave-active .bg-white {
  transition: transform 0.25s ease;
}
.modal-enter-from .bg-white,
.modal-leave-to .bg-white {
  transform: scale(0.95);
}

/* Marcador y etiqueta Strip Center en el mapa */
:deep(.strip-center-tooltip) {
  background: #1e3a8a !important;
  color: white !important;
  border: none !important;
  padding: 6px 12px !important;
  font-weight: 600 !important;
  font-size: 13px !important;
  border-radius: 8px !important;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2) !important;
}
:deep(.strip-center-tooltip::before) {
  border-top-color: #1e3a8a !important;
}
</style>
