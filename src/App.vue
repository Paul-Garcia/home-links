<script setup>
import { computed, onMounted, ref } from 'vue'

const AUTH_STORAGE_KEY = 'home-links-unlocked'

const title = ref('DevOps Links')
const subtitle = ref('')
const categories = ref([])
const search = ref('')
const loadError = ref(false)
const loading = ref(true)

const auth = ref(null)
const unlocked = ref(false)
const passwordInput = ref('')
const authError = ref(false)

async function sha256Hex(text) {
  const bytes = new TextEncoder().encode(text)
  const digest = await crypto.subtle.digest('SHA-256', bytes)
  return [...new Uint8Array(digest)].map((b) => b.toString(16).padStart(2, '0')).join('')
}

async function loadLinks() {
  try {
    const res = await fetch(`${import.meta.env.BASE_URL}links.json?t=${Date.now()}`)
    if (!res.ok) throw new Error(res.statusText)
    const data = await res.json()
    title.value = data.title ?? title.value
    subtitle.value = data.subtitle ?? ''
    categories.value = data.categories ?? []
    auth.value = data.auth?.enabled ? data.auth : null

    if (auth.value) {
      unlocked.value = localStorage.getItem(AUTH_STORAGE_KEY) === auth.value.passwordHash
    } else {
      unlocked.value = true
    }
  } catch (err) {
    console.error('Impossible de charger links.json', err)
    loadError.value = true
  } finally {
    loading.value = false
  }
}

onMounted(loadLinks)

async function submitPassword() {
  authError.value = false
  const hash = await sha256Hex(passwordInput.value)

  if (hash === auth.value.passwordHash) {
    localStorage.setItem(AUTH_STORAGE_KEY, hash)
    unlocked.value = true
  } else {
    authError.value = true
  }

  passwordInput.value = ''
}

function lock() {
  localStorage.removeItem(AUTH_STORAGE_KEY)
  unlocked.value = false
}

const filteredCategories = computed(() => {
  const q = search.value.trim().toLowerCase()
  if (!q) return categories.value

  return categories.value
    .map((cat) => ({
      ...cat,
      links: cat.links.filter((link) =>
        [link.name, link.description, link.url]
          .filter(Boolean)
          .some((field) => field.toLowerCase().includes(q))
      ),
    }))
    .filter((cat) => cat.links.length > 0)
})

const totalLinks = computed(() =>
  categories.value.reduce((sum, cat) => sum + cat.links.length, 0)
)

function isImageIcon(icon) {
  return typeof icon === 'string' && /^(https?:\/\/|\/|\.\/|\.\.\/)/.test(icon)
}
</script>

<template>
  <div class="page">
    <p v-if="loading" class="state-msg loading-msg">Chargement…</p>

    <div v-else-if="auth && !unlocked" class="lock-screen">
      <form class="lock-card" @submit.prevent="submitPassword">
        <div class="lock-icon">
          <svg viewBox="0 0 24 24" width="26" height="26">
            <path
              d="M6 10V8a6 6 0 1 1 12 0v2M5 10h14a1 1 0 0 1 1 1v9a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1v-9a1 1 0 0 1 1-1Z"
              fill="none"
              stroke="currentColor"
              stroke-width="1.6"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </div>
        <h1 class="lock-title">{{ title }}</h1>
        <p class="lock-subtitle">Accès protégé par mot de passe</p>
        <input
          v-model="passwordInput"
          type="password"
          placeholder="Mot de passe"
          class="search-input"
          autofocus
        />
        <button type="submit" class="unlock-btn">Déverrouiller</button>
        <p v-if="authError" class="auth-error">Mot de passe incorrect.</p>
      </form>
    </div>

    <template v-else>
      <header class="hero">
        <h1>{{ title }}</h1>
        <p v-if="subtitle" class="subtitle">{{ subtitle }}</p>

        <div class="search-wrap">
          <svg class="search-icon" viewBox="0 0 24 24" width="18" height="18">
            <path
              d="M21 21l-4.35-4.35m1.35-5.15a7 7 0 1 1-14 0 7 7 0 0 1 14 0Z"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
            />
          </svg>
          <input
            v-model="search"
            type="text"
            placeholder="Rechercher un outil… (ex: vault, harbor)"
            class="search-input"
          />
        </div>
      </header>

      <main class="content">
        <p v-if="loadError" class="state-msg error">
          Impossible de charger <code>links.json</code>. Vérifie que le fichier existe dans
          <code>public/links.json</code>.
        </p>

        <p v-else-if="totalLinks === 0" class="state-msg">
          Aucun lien configuré. Ajoute des entrées dans <code>public/links.json</code>.
        </p>

        <p v-else-if="filteredCategories.length === 0" class="state-msg">
          Aucun résultat pour « {{ search }} ».
        </p>

        <section v-for="cat in filteredCategories" :key="cat.name" class="category">
          <h2 class="category-title">{{ cat.name }}</h2>
          <div class="grid">
            <a
              v-for="link in cat.links"
              :key="link.url"
              :href="link.url"
              target="_blank"
              rel="noopener noreferrer"
              class="card"
              :style="{ '--accent-color': link.color || '#7c3aed' }"
            >
              <div class="card-icon" :class="{ 'card-icon--image': isImageIcon(link.icon) }">
                <img
                  v-if="isImageIcon(link.icon)"
                  :src="link.icon"
                  :alt="link.name"
                  loading="lazy"
                />
                <svg v-else viewBox="0 0 24 24" width="20" height="20">
                  <path
                    d="M9 15 15 9M10 6l.9-.9a4.24 4.24 0 1 1 6 6l-.9.9M14 18l-.9.9a4.24 4.24 0 1 1-6-6l.9-.9"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="1.8"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </div>
              <div class="card-body">
                <div class="card-name">{{ link.name }}</div>
                <div v-if="link.description" class="card-desc">{{ link.description }}</div>
              </div>
              <svg class="card-arrow" viewBox="0 0 24 24" width="18" height="18">
                <path
                  d="M7 17 17 7M8 7h9v9"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.8"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
            </a>
          </div>
        </section>
      </main>

      <footer class="footer">
        <span>Si tu veux ajouter un lien bah non faut modifier sur le repo.</span>
        <button v-if="auth" type="button" class="lock-link" @click="lock">Verrouiller</button>
      </footer>
    </template>
  </div>
</template>

<style scoped>
.page {
  max-width: 1100px;
  margin: 0 auto;
  padding: 3rem 1.5rem 4rem;
}

.hero {
  text-align: center;
  margin-bottom: 3rem;
}

.hero h1 {
  font-size: clamp(2rem, 5vw, 2.75rem);
  margin: 0 0 0.5rem;
  background: linear-gradient(120deg, var(--accent), var(--accent-2));
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-weight: 800;
  letter-spacing: -0.02em;
}

.subtitle {
  color: var(--text-dim);
  margin: 0 0 2rem;
  font-size: 1.05rem;
}

.search-wrap {
  position: relative;
  max-width: 480px;
  margin: 0 auto;
}

.search-icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  color: var(--text-dim);
  pointer-events: none;
}

.search-input {
  width: 100%;
  padding: 0.85rem 1rem 0.85rem 2.6rem;
  border-radius: 12px;
  border: 1px solid var(--border);
  background: var(--bg-soft);
  color: var(--text);
  font-size: 1rem;
  outline: none;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.search-input::placeholder {
  color: var(--text-dim);
}

.search-input:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(124, 58, 237, 0.25);
}

.state-msg {
  text-align: center;
  color: var(--text-dim);
  padding: 2rem 0;
}

.state-msg.error {
  color: #f87171;
}

.state-msg code {
  background: var(--bg-soft);
  padding: 0.15rem 0.4rem;
  border-radius: 6px;
  border: 1px solid var(--border);
}

.category {
  margin-bottom: 2.5rem;
}

.category-title {
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: var(--text-dim);
  margin: 0 0 1rem;
  font-weight: 700;
}

.grid {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.card {
  display: flex;
  align-items: flex-start;
  gap: 0.9rem;
  padding: 1.1rem 1.2rem;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 14px;
  transition: transform 0.15s ease, background 0.15s ease, border-color 0.15s ease,
    box-shadow 0.15s ease;
  position: relative;
  overflow: hidden;
}

.card::before {
  content: '';
  position: absolute;
  inset: 0;
  background: var(--accent-color);
  opacity: 0;
  transition: opacity 0.15s ease;
}

.card:hover {
  transform: translateY(-3px);
  background: var(--card-hover);
  border-color: var(--accent-color);
  box-shadow: 0 10px 30px -12px rgba(0, 0, 0, 0.6);
}

.card:hover::before {
  opacity: 0.06;
}

.card-icon {
  width: 2.6rem;
  height: 2.6rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  flex-shrink: 0;
  background: color-mix(in srgb, var(--accent-color) 16%, transparent);
  color: var(--accent-color);
}

.card-icon--image {
  background: #fff;
  padding: 6px;
}

.card-icon--image img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.card-body {
  min-width: 0;
  flex: 1;
}

.card-name {
  font-weight: 600;
  font-size: 1.02rem;
  margin-bottom: 0.15rem;
}

.card-desc {
  font-size: 0.85rem;
  color: var(--text-dim);
  overflow-wrap: break-word;
  line-height: 1.35;
}

.card-arrow {
  color: var(--text-dim);
  opacity: 0;
  transform: translate(-4px, 4px);
  transition: opacity 0.15s ease, transform 0.15s ease;
  flex-shrink: 0;
}

.card:hover .card-arrow {
  opacity: 1;
  transform: translate(0, 0);
  color: var(--accent-color);
}

.footer {
  text-align: center;
  margin-top: 3rem;
  color: var(--text-dim);
  font-size: 0.85rem;
}

.footer code {
  background: var(--bg-soft);
  padding: 0.15rem 0.4rem;
  border-radius: 6px;
  border: 1px solid var(--border);
}

.footer {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.6rem;
}

.lock-link {
  background: none;
  border: 1px solid var(--border);
  color: var(--text-dim);
  padding: 0.35rem 0.9rem;
  border-radius: 8px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;
}

.lock-link:hover {
  border-color: var(--accent);
  color: var(--text);
}

.loading-msg {
  padding-top: 4rem;
}

.lock-screen {
  min-height: 70vh;
  display: flex;
  align-items: center;
  justify-content: center;
}

.lock-card {
  width: 100%;
  max-width: 360px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.9rem;
  padding: 2.5rem 2rem;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 16px;
  text-align: center;
}

.lock-icon {
  width: 3.2rem;
  height: 3.2rem;
  border-radius: 50%;
  background: color-mix(in srgb, var(--accent) 18%, transparent);
  color: var(--accent);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.lock-title {
  font-size: 1.5rem;
  margin: 0;
  background: linear-gradient(120deg, var(--accent), var(--accent-2));
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-weight: 800;
}

.lock-subtitle {
  color: var(--text-dim);
  font-size: 0.95rem;
  margin: 0;
}

.lock-card .search-input {
  padding: 0.85rem 1rem;
}

.unlock-btn {
  width: 100%;
  padding: 0.75rem;
  border-radius: 10px;
  border: none;
  background: linear-gradient(120deg, var(--accent), var(--accent-2));
  color: #fff;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  text-align: center;
  transition: opacity 0.15s ease, transform 0.15s ease;
}

.unlock-btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

.auth-error {
  color: #f87171;
  font-size: 0.85rem;
  margin: 0;
}
</style>
