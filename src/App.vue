<script setup>
import { ref, onMounted } from 'vue'
import logoCugat from './assets/logos/cugat.png'
import logoKfc from './assets/logos/kfc.png'
import logoPapaJohns from './assets/logos/papajohns.png'
import logoCruzVerde from './assets/logos/cruzverde.png'

// Contadores animados
const terreno = ref(0)
const estacionamientos = ref(0)
const locales = ref(0)
const countDuration = 2000
const countSteps = 60

function animateValue(ref, end, suffix = '') {
  const start = 0
  const startTime = performance.now()
  const step = (now) => {
    const elapsed = now - startTime
    const progress = Math.min(elapsed / countDuration, 1)
    const easeOut = 1 - Math.pow(1 - progress, 3)
    ref.value = Math.floor(start + (end - start) * easeOut)
    if (progress < 1) requestAnimationFrame(step)
    else ref.value = end
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

const whatsappNumber = '56912345678' // Reemplazar con número real
const whatsappBase = `https://wa.me/${whatsappNumber}`

// Ruta del video: coloque drone.mp4 en public/videos/drone.mp4
const videoSrc = '/videos/drone.mp4'
const videoError = ref(false)
function onVideoError() {
  videoError.value = true
}

// Observador para animaciones de entrada
onMounted(() => {
  animateValue(terreno, 11800)
  animateValue(estacionamientos, 143)
  animateValue(locales, 15)

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          entry.target.classList.add('visible')
        }
      })
    },
    { threshold: 0.1, rootMargin: '0px 0px -50px 0px' }
  )
  document.querySelectorAll('.animate-on-scroll').forEach((el) => observer.observe(el))
})
</script>

<template>
  <div class="min-h-screen bg-white text-[#1e3a8a]">
    <!-- Hero con video -->
    <header class="relative h-[90vh] min-h-[500px] flex items-center justify-center overflow-hidden">
      <!-- Fondo por si no hay video: gradiente Talca/comercial -->
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
      <!-- Overlay suave para que se vea el video cuando exista -->
      <div class="absolute inset-0 bg-black/40" />
      <div class="relative z-10 px-4 text-center text-white max-w-4xl mx-auto">
        <p class="text-lircay-orange font-semibold tracking-widest uppercase text-sm md:text-base opacity-0 animate-fade-in-up animate-delay-100">
          Strip Center Lircay
        </p>
        <h1 class="text-3xl sm:text-4xl md:text-5xl lg:text-6xl font-bold leading-tight mt-2 opacity-0 animate-fade-in-up animate-delay-200">
          Oportunidad de Inversión en el Corazón Comercial de Talca
        </h1>
        <p class="mt-4 text-lg md:text-xl text-white/90 opacity-0 animate-fade-in-up animate-delay-300">
          Ubicación privilegiada. Marcas ancla consolidadas. Alto flujo vehicular.
        </p>
        <a
          href="#locales"
          class="inline-block mt-8 px-8 py-4 bg-[#f37021] text-white font-semibold rounded-lg hover:bg-[#e06518] transition-all duration-300 shadow-lg hover:shadow-xl opacity-0 animate-fade-in-up animate-delay-400"
        >
          Ver locales disponibles
        </a>
      </div>
    </header>

    <!-- Cifras clave -->
    <section class="py-16 md:py-24 bg-[#1e3a8a] text-white">
      <div class="max-w-6xl mx-auto px-4">
        <h2 class="text-2xl md:text-3xl font-bold text-center mb-12 text-white/95">
          Cifras que respaldan tu inversión
        </h2>
        <div class="grid grid-cols-1 md:grid-cols-3 gap-8 md:gap-12">
          <div class="text-center animate-on-scroll opacity-0 translate-y-6 transition-all duration-700">
            <p class="text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">
              {{ terreno.toLocaleString('es-CL') }}
            </p>
            <p class="text-lg md:text-xl mt-1 text-white/90">m² de terreno</p>
          </div>
          <div class="text-center animate-on-scroll opacity-0 translate-y-6 transition-all duration-700" style="transition-delay: 100ms">
            <p class="text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">
              {{ estacionamientos }}
            </p>
            <p class="text-lg md:text-xl mt-1 text-white/90">estacionamientos</p>
          </div>
          <div class="text-center animate-on-scroll opacity-0 translate-y-6 transition-all duration-700" style="transition-delay: 200ms">
            <p class="text-4xl md:text-5xl font-bold text-[#f37021] tabular-nums">
              {{ locales }}
            </p>
            <p class="text-lg md:text-xl mt-1 text-white/90">locales totales</p>
          </div>
        </div>
      </div>
    </section>

    <!-- Marcas ancla - slider infinito -->
    <section class="py-12 md:py-16 bg-gray-50 border-y border-gray-200">
      <h2 class="text-xl md:text-2xl font-bold text-[#1e3a8a] text-center mb-8 px-4">
        Marcas ancla
      </h2>
      <div class="overflow-hidden">
        <div class="slider-track gap-12 md:gap-20 px-4">
          <template v-for="(_, i) in 2" :key="'set-' + i">
            <div
              v-for="m in marcas"
              :key="m.nombre + i"
              class="flex-shrink-0 flex items-center justify-center w-48 md:w-56 h-20 md:h-24 bg-white rounded-xl shadow-md border border-gray-100 px-6"
            >
              <img
                v-if="m.logo"
                :src="m.logo"
                :alt="m.nombre"
                class="max-h-14 md:max-h-16 w-auto min-h-[2.5rem] object-contain block"
                @error="($event.target).style.display='none'; ($event.target).nextElementSibling?.classList.remove('hidden')"
              />
              <span v-if="m.logo" class="hidden font-semibold text-[#1e3a8a] text-center text-sm md:text-base">{{ m.nombre }}</span>
              <span v-else class="font-semibold text-[#1e3a8a] text-center text-sm md:text-base">{{ m.nombre }}</span>
            </div>
          </template>
        </div>
      </div>
    </section>

    <!-- Tabla de locales -->
    <section id="locales" class="py-16 md:py-24 px-4">
      <div class="max-w-4xl mx-auto">
        <h2 class="text-2xl md:text-3xl font-bold text-[#1e3a8a] text-center mb-4">
          Locales disponibles
        </h2>
        <p class="text-center text-gray-600 mb-10">
          Consulta disponibilidad y condiciones. Cotiza sin compromiso.
        </p>
        <ul class="space-y-4">
          <li
            v-for="(loc, index) in localesDisponibles"
            :key="loc.id"
            class="animate-on-scroll opacity-0 translate-y-4 transition-all duration-500 flex flex-col sm:flex-row sm:items-center justify-between gap-4 p-4 md:p-5 rounded-xl bg-gray-50 hover:bg-gray-100 border border-gray-200"
            :style="{ transitionDelay: `${index * 80}ms` }"
          >
            <div class="flex items-center gap-4">
              <span class="text-2xl font-bold text-[#1e3a8a]">Local {{ loc.id }}</span>
              <span class="text-gray-600">{{ loc.m2 }} m²</span>
            </div>
            <a
              :href="`${whatsappBase}?text=${encodeURIComponent('Hola, me interesa cotizar el Local ' + loc.id + ' (' + loc.m2 + ' m²) en Strip Center Lircay.')}`"
              target="_blank"
              rel="noopener"
              class="inline-flex items-center justify-center px-6 py-3 bg-[#f37021] text-white font-semibold rounded-lg hover:bg-[#e06518] transition-colors shrink-0"
            >
              Cotizar
            </a>
          </li>
        </ul>
      </div>
    </section>

    <!-- Ubicación -->
    <section class="py-16 md:py-24 bg-[#1e3a8a] text-white">
      <div class="max-w-4xl mx-auto px-4 text-center">
        <h2 class="text-2xl md:text-3xl font-bold mb-6">
          Ubicación estratégica
        </h2>
        <p class="text-lg md:text-xl text-white/90 leading-relaxed">
          Conectividad directa con <strong class="text-[#f37021]">Av. Lircay</strong> y
          <strong class="text-[#f37021]">Ruta 5 Sur</strong>. Alto tráfico vehicular y visibilidad
          excepcional para tu negocio en el corazón comercial de Talca.
        </p>
        <p class="mt-4 text-white/80">
          Strip Center Lircay — Talca, Región del Maule
        </p>
      </div>
    </section>

    <!-- Footer -->
    <footer class="py-8 bg-gray-900 text-white/80 text-center text-sm">
      <p>Strip Center Lircay — Oportunidad de inversión comercial en Talca.</p>
    </footer>
  </div>
</template>

<style scoped>
.animate-on-scroll.visible {
  opacity: 1;
  transform: translateY(0);
}
</style>
