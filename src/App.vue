<script setup>
import { ref, computed, onMounted } from 'vue'

// Variables reactivas
const paises = ref([])
const busqueda = ref('')
const consulta = ref('Canada')
const cargando = ref(false)
const error = ref('')
const avisoDemo = ref(false)

// Permite comparar palabras sin distinguir mayúsculas ni acentos.
function normalizar(texto) {
  return String(texto ?? '')
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .toLowerCase()
    .trim()
}

// El filtro se aplica a los países que ya fueron obtenidos.
const paisesFiltrados = computed(() => {
  const filtro = normalizar(busqueda.value)

  return paises.value.filter((pais) => {
    return normalizar(pais.nombre).includes(filtro) ||
      normalizar(pais.region).includes(filtro)
  })
})

// Petición asíncrona a REST Countries
async function obtenerPaises() {
  cargando.value = true
  error.value = ''
  avisoDemo.value = false
  paises.value = []

  try {
    const url = new URL(
      'https://api.restcountries.com/countries/v5'
    )

    url.searchParams.set('q', consulta.value.trim() || 'Canada')
    url.searchParams.set('pretty', '1')

    const respuesta = await fetch(url, {
  headers: {
    Authorization: 'Bearer rc_live_f92f0ecc003d475982737539d6a5c586',
  },
})

    if (!respuesta.ok) {
  const detalle = await respuesta.text()

  throw new Error(
    `Estado ${respuesta.status}. Detalle: ${detalle}`
  )
}

    const datos = await respuesta.json()
    const lista = datos.data?.objects

    if (!Array.isArray(lista)) {
      console.error('Respuesta recibida:', datos)
      throw new Error('La API devolvió un formato diferente al esperado.')
    }

    avisoDemo.value = Boolean(datos.data?._demo)

    // Adaptamos los datos de la API a las tarjetas.
    paises.value = lista.map((pais) => ({
      id: pais.codes?.alpha_3 ?? pais.codes?.alpha_2 ??
        pais.uuid ?? pais.names?.common,
      nombre: pais.names?.common ?? 'Nombre no disponible',
      bandera: pais.flag?.url_png ?? pais.flag?.url_svg ?? '',
      emoji: pais.flag?.emoji ?? '🌍',
      capital: pais.capitals?.map((capital) => capital.name)
        .join(', ') || 'Sin capital registrada',
      region: pais.region ?? 'Región no disponible',
      poblacion: pais.population ?? null,
    }))
  } catch (problema) {
    console.error(problema)
    error.value =
      `No fue posible obtener los datos de la API. ${problema.message}`
  } finally {
    cargando.value = false
  }
}

// La primera llamada se ejecuta al montar el componente.
onMounted(() => {
  obtenerPaises()
})

function formatearPoblacion(poblacion) {
  return typeof poblacion === 'number'
    ? poblacion.toLocaleString('es-MX')
    : 'No disponible'
}
</script>

<template>
  <main class="contenedor">
    <header class="encabezado">
      <span class="etiqueta">Vue 3 + REST Countries</span>
      <h1> Explorador de Países</h1>
      <p>Consulta países y descubre información sobre ellos.</p>
    </header>

    <section class="panel">
      <form @submit.prevent="obtenerPaises">
        <label for="consulta">Consultar un país en la API</label>

        <div class="fila">
          <input
            id="consulta"
            v-model="consulta"
            placeholder="Ejemplo: Canada"
            :disabled="cargando"
          >

          <button type="submit" :disabled="cargando">
            {{ cargando ? 'Consultando...' : 'Consultar API' }}
          </button>
        </div>
      </form>

      <label for="busqueda">Filtrar los resultados por nombre o región</label>

      <input
        id="busqueda"
        v-model="busqueda"
        type="search"
        placeholder="Escribe un nombre o región"
      >

      <p class="ayuda">
        La consulta obtiene datos de internet. El filtro busca dentro
        de los resultados recibidos.
      </p>
    </section>

    <p v-if="cargando" class="mensaje" role="status">
       Cargando países...
    </p>

    <section v-else-if="error" class="mensaje error" role="alert">
      <p>{{ error }}</p>
      <button @click="obtenerPaises">Intentar nuevamente</button>
    </section>

    <template v-else>
      <p v-if="avisoDemo" class="mensaje">
        La API está proporcionando una respuesta de demostración.
      </p>

      <p class="contador">
        Países mostrados: {{ paisesFiltrados.length }}
      </p>

      <p v-if="paisesFiltrados.length === 0" class="mensaje">
        No se encontraron países para mostrar.
        Prueba otro filtro o realiza una nueva consulta.
      </p>

      <section v-else class="tarjetas" aria-label="Lista de países">
        <article
          v-for="pais in paisesFiltrados"
          :key="pais.id"
          class="tarjeta"
        >
          <img
            v-if="pais.bandera"
            :src="pais.bandera"
            :alt="'Bandera de ' + pais.nombre"
            class="bandera"
          >

          <div v-else class="bandera alternativa">
            {{ pais.emoji }}
          </div>

          <div class="informacion">
            <h2>{{ pais.nombre }}</h2>
            <p><strong>Capital:</strong> {{ pais.capital }}</p>
            <p><strong>Región:</strong> {{ pais.region }}</p>
            <p>
              <strong>Población:</strong>
              {{ formatearPoblacion(pais.poblacion) }}
            </p>
          </div>
        </article>
      </section>
    </template>

    <footer>
      Segundo proyecto · Unidad 3 · Consumo de API con Vue 3
    </footer>
  </main>
</template>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f6f0fb;
  color: #352846;
}

button,
input {
  font: inherit;
}

.contenedor {
  max-width: 1100px;
  margin: auto;
  padding: 32px 20px;
}

.encabezado {
  text-align: center;
  margin-bottom: 28px;
}

.etiqueta {
  display: inline-block;
  padding: 8px 14px;
  background: #e8d8f6;
  border-radius: 20px;
  color: #654087;
}

h1 {
  color: #673c91;
  font-size: clamp(28px, 5vw, 42px);
}

.panel {
  background: white;
  padding: 24px;
  border-radius: 18px;
  box-shadow: 0 5px 20px #45236112;
}

label {
  display: block;
  margin-bottom: 10px;
  font-weight: bold;
}

.fila {
  display: flex;
  gap: 12px;
  margin-bottom: 22px;
}

input {
  width: 100%;
  min-width: 0;
  padding: 13px;
  border: 1px solid #cbb6df;
  border-radius: 10px;
  background: #fdfaff;
}

input:focus {
  outline: 2px solid #9664c1;
}

button {
  padding: 12px 20px;
  border: none;
  border-radius: 10px;
  background: #7950a3;
  color: white;
  cursor: pointer;
}

button:disabled {
  opacity: 0.65;
  cursor: wait;
}

.ayuda,
.contador {
  color: #706079;
}

.ayuda {
  font-size: 14px;
  line-height: 1.5;
}

.mensaje {
  padding: 22px;
  background: #eaddf5;
  border-radius: 12px;
  text-align: center;
}

.error {
  background: #ffe5ec;
  color: #8c2443;
}

.tarjetas {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 22px;
}

.tarjeta {
  background: white;
  border-radius: 18px;
  overflow: hidden;
  box-shadow: 0 5px 18px #45236112;
}

.bandera {
  display: block;
  width: 100%;
  height: 170px;
  object-fit: contain;
  background: #eee7f5;
}

.alternativa {
  display: grid;
  place-items: center;
  font-size: 70px;
}

.informacion {
  padding: 20px;
}

.informacion h2 {
  margin-top: 0;
  color: #673c91;
}

.informacion p {
  line-height: 1.5;
}

footer {
  margin-top: 36px;
  text-align: center;
  color: #706079;
  font-size: 14px;
}

@media (max-width: 600px) {
  .fila {
    flex-direction: column;
  }
}
</style>