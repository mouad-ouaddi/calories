<script setup>
import { ref, computed, onMounted } from 'vue'
import UiSelect from './components/UiSelect.vue'

const age = ref('')
const sex = ref('homme')
const height = ref('')
const weight = ref('')
const bodyFat = ref('')
const activity = ref('1.55')
const activityOptions = [
  { value: '1.2', label: 'Sédentaire (peu ou pas d’exercice)' },
  { value: '1.375', label: 'Léger (15–30 min / jour)' },
  { value: '1.55', label: 'Modéré (30–60 min / jour)' },
  { value: '1.725', label: 'Intense (60–120 min / jour)' },
  { value: '1.9', label: 'Très intense (2+ heures / jour)' },
]
const hasCalculated = ref(false)
const tab = ref('home')

const theme = ref('dark')
const user = ref(null)
const showAuth = ref(false)
const authMode = ref('login')
const authEmail = ref('')
const authPassword = ref('')
const authName = ref('')
const showProfileMenu = ref(false)
const authError = ref('')
const authLoading = ref(false)
const currentPage = ref('home')
const goHome = () => { currentPage.value = 'home' }
const goRun = () => { currentPage.value = 'run' }
const goal = ref('maintain')
const macroPreset = ref('equilibre')
const macroPresets = {
  equilibre: { label: 'Équilibré', proteinPerKg: 1.6, fatRatio: 0.25 },
  hyperproteine: { label: 'Hyperprotéiné', proteinPerKg: 2.0, fatRatio: 0.25 },
  pauvre_en_glucides: { label: 'Pauvre en glucides', proteinPerKg: 2.2, fatRatio: 0.35 },
}
const getAvgForKey = (key) => {
  const rows = results.value || []
  if (!rows.length) return 0
  const sum = rows.reduce((a, r) => a + (r[key] || 0), 0)
  return Math.round(sum / rows.length)
}
const targetCalories = computed(() => getAvgForKey(goal.value))
const macros = computed(() => {
  const calories = targetCalories.value
  const W = parseNum(weight.value)
  const preset = macroPresets[macroPreset.value] || macroPresets.equilibre
  const proteinGrams = Math.round(preset.proteinPerKg * W)
  const proteinCalories = proteinGrams * 4
  const fatCalories = Math.round(calories * preset.fatRatio)
  const fatGrams = Math.round(fatCalories / 9)
  const carbsCalories = Math.max(0, calories - proteinCalories - fatCalories)
  const carbsGrams = Math.round(carbsCalories / 4)
  return { calories, proteinGrams, fatGrams, carbsGrams }
})
const planSaveMsg = ref('')
const savePlan = () => {
  planSaveMsg.value = ''
  const entry = {
    date: new Date().toISOString(),
    goal: goal.value,
    preset: macroPreset.value,
    calories: macros.value.calories,
    proteinGrams: macros.value.proteinGrams,
    fatGrams: macros.value.fatGrams,
    carbsGrams: macros.value.carbsGrams,
    weight: parseNum(weight.value),
  }
  const raw = localStorage.getItem('saved_plans')
  let arr = []
  if (raw) {
    try { arr = JSON.parse(raw) } catch {}
  }
  arr.push(entry)
  localStorage.setItem('saved_plans', JSON.stringify(arr))
  planSaveMsg.value = 'Plan enregistré'
  loadPlans()
}
const savedPlans = ref([])
const loadPlans = () => {
  const raw = localStorage.getItem('saved_plans')
  if (!raw) { savedPlans.value = []; return }
  try {
    const v = JSON.parse(raw)
    savedPlans.value = Array.isArray(v) ? v.slice().reverse() : []
  } catch {
    savedPlans.value = []
  }
}
const showPlanPreview = ref(false)
const previewPlan = ref(null)
const openPlanPreview = (p) => { previewPlan.value = p; showPlanPreview.value = true }
const closePlanPreview = () => { showPlanPreview.value = false; previewPlan.value = null }
const runActive = ref(false)
const runAvatarLane = ref(1)
const runItems = ref([])
const runSpeed = ref(2)
const runDistance = ref(0)
const runHealth = ref(0)
const runCombo = ref(0)
const runLastJunkAt = ref(0)
const runBest = ref(0)
const runLevel = computed(() => {
  const h = runHealth.value
  if (h >= 200) return 4
  if (h >= 120) return 3
  if (h >= 50) return 2
  return 1
})
const runEmoji = computed(() => {
  const l = runLevel.value
  return l === 4 ? '💪' : l === 3 ? '🏃' : l === 2 ? '🙂' : '😐'
})
const runLevelImageUrl = computed(() => '/assets/runner/level-' + runLevel.value + '.png')
const runAvatarSrc = computed(() => runAvatarUrl.value || runLevelImageUrl.value)
const runAvatarUrl = ref('')
const loadRunAvatarFromBackend = async () => {
  try {
    const email = user.value?.email || ''
    const q = email ? ('?email=' + encodeURIComponent(email)) : ''
    const r = await fetch('/api/avatar' + q)
    if (r.ok) {
      const j = await r.json()
      if (j && j.url) runAvatarUrl.value = String(j.url)
    }
  } catch {}
}
let runTickHandle = null
let runSpawnHandle = null
let runContainerHeight = 260
const healthyIcons = ['🍚','🍗','🥦','🍎','💧']
const junkIcons = ['🍔','🍕','🥤','🍟']
const loadRunBest = () => {
  const raw = localStorage.getItem('healthy_run_best')
  if (raw) {
    const n = parseInt(raw)
    runBest.value = Number.isFinite(n) ? n : 0
  }
}
const addItem = () => {
  const lane = Math.floor(Math.random() * 3)
  const isHealthy = Math.random() < 0.6
  const icon = isHealthy ? healthyIcons[Math.floor(Math.random()*healthyIcons.length)] : junkIcons[Math.floor(Math.random()*junkIcons.length)]
  const id = Date.now() + Math.random()
  runItems.value = [...runItems.value, { id, lane, y: -20, healthy: isHealthy, icon }]
}
const moveLane = (dir) => {
  const next = Math.min(2, Math.max(0, runAvatarLane.value + dir))
  runAvatarLane.value = next
}
const onKeydown = (e) => {
  if (!runActive.value) return
  if (e.key === 'ArrowLeft') moveLane(-1)
  if (e.key === 'ArrowRight') moveLane(1)
}
const applyItem = (it) => {
  if (it.healthy) {
    runHealth.value += 10
    runCombo.value += 1
    if (runCombo.value >= 3) { runHealth.value += 30; runCombo.value = 0 }
    runSpeed.value = Math.min(6, runSpeed.value + 0.3)
  } else {
    runHealth.value -= 10
    runCombo.value = 0
    runSpeed.value = Math.max(1, runSpeed.value - 0.4)
    runLastJunkAt.value = Date.now()
  }
}
const tick = () => {
  runItems.value = runItems.value.map(it => ({ ...it, y: it.y + runSpeed.value*3 }))
  const catchY = runContainerHeight - 60
  const caught = []
  const remain = []
  for (const it of runItems.value) {
    if (it.y >= catchY) {
      if (it.lane === runAvatarLane.value) caught.push(it)
    } else {
      remain.push(it)
    }
  }
  for (const it of caught) applyItem(it)
  runItems.value = remain
  runDistance.value += Math.round(runSpeed.value)
  if (runLastJunkAt.value > 0) {
    const sec = (Date.now() - runLastJunkAt.value) / 1000
    if (sec >= 20) { runSpeed.value = Math.min(6.5, runSpeed.value + 0.5); runLastJunkAt.value = 0 }
  }
}
const startRun = async () => {
  if (runActive.value) return
  runActive.value = true
  runAvatarLane.value = 1
  runItems.value = []
  runSpeed.value = 2
  runDistance.value = 0
  runHealth.value = 0
  runCombo.value = 0
  runLastJunkAt.value = 0
  loadRunBest()
  await loadRunAvatarFromBackend()
  window.addEventListener('keydown', onKeydown)
  runTickHandle = setInterval(tick, 60)
  runSpawnHandle = setInterval(addItem, 1100)
}
const stopRun = () => {
  if (!runActive.value) return
  runActive.value = false
  window.removeEventListener('keydown', onKeydown)
  if (runTickHandle) clearInterval(runTickHandle)
  if (runSpawnHandle) clearInterval(runSpawnHandle)
  runTickHandle = null
  runSpawnHandle = null
  if (runDistance.value > runBest.value) {
    runBest.value = runDistance.value
    localStorage.setItem('healthy_run_best', String(runBest.value))
  }
}
 
 
const applySavedPlan = (p) => {
  goal.value = p.goal
  macroPreset.value = p.preset
  weight.value = String(p.weight || '')
  planSaveMsg.value = 'Plan chargé'
}
const deletePlanAt = (i) => {
  const raw = localStorage.getItem('saved_plans')
  let arr = []
  if (raw) {
    try { arr = JSON.parse(raw) } catch {}
  }
  const idx = (arr.length - 1) - i
  if (idx >= 0 && idx < arr.length) {
    arr.splice(idx, 1)
    localStorage.setItem('saved_plans', JSON.stringify(arr))
  }
  loadPlans()
}
const showAccountEdit = ref(false)
const editName = ref('')
const editEmail = ref('')
const editCurrentPassword = ref('')
const editNewPassword = ref('')
const editConfirmPassword = ref('')
const editError = ref('')
const editLoading = ref(false)

const parseNum = (v) => {
  const n = typeof v === 'string' ? parseFloat(v) : v
  return Number.isFinite(n) ? n : 0
}

const bmrMifflin = computed(() => {
  const W = parseNum(weight.value)
  const H = parseNum(height.value)
  const A = parseNum(age.value)
  return sex.value === 'homme'
    ? 10 * W + 6.25 * H - 5 * A + 5
    : 10 * W + 6.25 * H - 5 * A - 161
})

const bmrHarris = computed(() => {
  const W = parseNum(weight.value)
  const H = parseNum(height.value)
  const A = parseNum(age.value)
  return sex.value === 'homme'
    ? 13.397 * W + 4.799 * H - 5.677 * A + 88.362
    : 9.247 * W + 3.098 * H - 4.330 * A + 447.593
})

const bmrKatch = computed(() => {
  const W = parseNum(weight.value)
  const f = parseNum(bodyFat.value)
  if (!f && f !== 0) return null
  const F = f / 100
  return 370 + 21.6 * (1 - F) * W
})

const factor = computed(() => parseNum(activity.value))

const toRound = (n) => Math.round(n)
const kcalPerKg = 7700
const deltas = {
  lose025: kcalPerKg * 0.25 / 7,
  lose05: kcalPerKg * 0.5 / 7,
  lose1: kcalPerKg * 1 / 7,
}

const results = computed(() => {
  const f = factor.value
  const makeRow = (label, bmr) => {
    const tdee = bmr * f
    return {
      label,
      maintain: toRound(tdee),
      lose025: toRound(tdee - deltas.lose025),
      lose05: toRound(tdee - deltas.lose05),
      lose1: toRound(tdee - deltas.lose1),
      gain025: toRound(tdee + deltas.lose025),
      gain05: toRound(tdee + deltas.lose05),
      gain1: toRound(tdee + deltas.lose1),
    }
  }
  const rows = []
  rows.push(makeRow('Mifflin-St Jeor', bmrMifflin.value))
  rows.push(makeRow('Harris-Benedict (révisée)', bmrHarris.value))
  if (bmrKatch.value != null) rows.push(makeRow('Katch-McArdle', bmrKatch.value))
  return rows
})

const calculate = () => {
  hasCalculated.value = true
}

const applyTheme = (t) => {
  document.documentElement.setAttribute('data-theme', t)
  localStorage.setItem('theme', t)
}
const toggleTheme = () => {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
  applyTheme(theme.value)
}
const themeIcon = computed(() => theme.value === 'dark' ? 'moon' : 'sun')

const avatarLabel = computed(() => {
  if (!user.value) return '👤'
  const n = user.value.name || user.value.email || ''
  const m = n.split(/[\s@._-]+/).filter(Boolean)
  const a = (m[0] || '').charAt(0).toUpperCase()
  const b = (m[1] || '').charAt(0).toUpperCase()
  return a + b || a || '👤'
})

const saveUser = (u) => {
  localStorage.setItem('app_user', JSON.stringify(u))
  user.value = u
}
const getUsers = () => {
  const raw = localStorage.getItem('app_users')
  if (!raw) return []
  try {
    const arr = JSON.parse(raw)
    return Array.isArray(arr) ? arr : []
  } catch {
    return []
  }
}
const setUsers = (arr) => {
  localStorage.setItem('app_users', JSON.stringify(arr))
}
const hashPassword = async (s) => {
  const enc = new TextEncoder().encode(s)
  const buf = await crypto.subtle.digest('SHA-256', enc)
  const view = new Uint8Array(buf)
  return Array.from(view).map(b => b.toString(16).padStart(2, '0')).join('')
}
const registerUser = async (email, name, pass) => {
  const users = getUsers()
  if (users.find(u => u.email.toLowerCase() === email.toLowerCase())) {
    throw new Error('Cet email est déjà utilisé')
  }
  const hash = await hashPassword(pass)
  const u = { email, name, pass: hash }
  users.push(u)
  setUsers(users)
  saveUser({ email, name })
}
const loginUser = async (email, pass) => {
  const users = getUsers()
  const found = users.find(u => u.email.toLowerCase() === email.toLowerCase())
  if (!found) throw new Error('Compte introuvable')
  const hash = await hashPassword(pass)
  if (found.pass !== hash) throw new Error('Mot de passe incorrect')
  saveUser({ email, name: found.name })
}
const loadUser = () => {
  const raw = localStorage.getItem('app_user')
  if (raw) {
    try { user.value = JSON.parse(raw) } catch {}
  }
}
const logout = () => {
  localStorage.removeItem('app_user')
  user.value = null
  showProfileMenu.value = false
}
const openAccountEdit = () => {
  editError.value = ''
  editLoading.value = false
  editName.value = user.value?.name || ''
  editEmail.value = user.value?.email || ''
  editCurrentPassword.value = ''
  editNewPassword.value = ''
  editConfirmPassword.value = ''
  showAccountEdit.value = true
}
const closeAccountEdit = () => {
  showAccountEdit.value = false
}
const onAuthSubmit = async () => {
  authError.value = ''
  const email = authEmail.value.trim()
  const pass = authPassword.value.trim()
  const name = authName.value.trim()
  if (!email) { authError.value = 'Email requis'; return }
  if (!pass) { authError.value = 'Mot de passe requis'; return }
  if (authMode.value === 'register' && name.length === 0) { authError.value = 'Nom requis'; return }
  try {
    authLoading.value = true
    if (authMode.value === 'register') {
      await registerUser(email, name, pass)
    } else {
      await loginUser(email, pass)
    }
    showAuth.value = false
    authEmail.value = ''
    authPassword.value = ''
    authName.value = ''
  } catch (e) {
    authError.value = e?.message || 'Erreur'
  } finally {
    authLoading.value = false
  }
}

const onAccountSave = async () => {
  editError.value = ''
  if (!user.value) { editError.value = 'Non connecté'; return }
  const prevEmail = (user.value.email || '').trim()
  const name = editName.value.trim()
  const email = editEmail.value.trim()
  const cur = editCurrentPassword.value
  const newp = editNewPassword.value
  const conf = editConfirmPassword.value
  const users = getUsers()
  const idx = users.findIndex(u => u.email.toLowerCase() === prevEmail.toLowerCase())
  if (idx === -1) { editError.value = 'Compte introuvable'; return }
  const existsOther = users.find(u => u.email.toLowerCase() === email.toLowerCase() && u.email.toLowerCase() !== prevEmail.toLowerCase())
  if (existsOther) { editError.value = 'Cet email est déjà utilisé'; return }
  try {
    editLoading.value = true
    const rec = users[idx]
    if (newp || conf || cur) {
      if (!cur) { throw new Error('Mot de passe actuel requis') }
      const curHash = await hashPassword(cur)
      if (rec.pass !== curHash) { throw new Error('Mot de passe actuel incorrect') }
      if (!newp) { throw new Error('Nouveau mot de passe requis') }
      if (newp !== conf) { throw new Error('La confirmation ne correspond pas') }
      rec.pass = await hashPassword(newp)
    }
    rec.name = name
    rec.email = email
    users[idx] = rec
    setUsers(users)
    saveUser({ email, name })
    showAccountEdit.value = false
  } catch (e) {
    editError.value = e?.message || 'Erreur'
  } finally {
    editLoading.value = false
  }
}

onMounted(() => {
  loadUser()
  const stored = localStorage.getItem('theme')
  theme.value = stored === 'light' ? 'light' : 'dark'
  applyTheme(theme.value)
  loadPlans()
})
</script>

<template>
  <div class="appbar">
    <div class="brand">caltracker</div>
    <div class="actions-bar">
      <button class="icon-btn" @click="toggleTheme" :aria-label="theme === 'dark' ? 'Mode clair' : 'Mode sombre'">
        <svg v-if="themeIcon === 'moon'" width="20" height="20" viewBox="0 0 24 24"><path fill="currentColor" d="M21 12.79A9 9 0 1 1 11.21 3a7 7 0 0 0 9.79 9.79Z"/></svg>
        <svg v-else width="20" height="20" viewBox="0 0 24 24"><path fill="currentColor" d="M12 18a6 6 0 1 0 0-12a6 6 0 0 0 0 12Zm0 4a1 1 0 0 1-1-1v-1a1 1 0 1 1 2 0v1a1 1 0 0 1-1 1Zm0-18a1 1 0 0 1-1-1V2a1 1 0 1 1 2 0v1a1 1 0 0 1-1 1Zm10 8a1 1 0 0 1-1-1v-1a1 1 0 1 1 2 0v1a1 1 0 0 1-1 1ZM3 12a1 1 0 0 1-1-1v-1a1 1 0 1 1 2 0v1a1 1 0 0 1-1 1Zm15.66 7.66a1 1 0 0 1-1.41 0l-.71-.71a1 1 0 1 1 1.41-1.41l.71.71a1 1 0 0 1 0 1.41ZM6.46 6.46a1 1 0 0 1-1.41 0l-.71-.71A1 1 0 0 1 5.75 3.64l.71.71a1 1 0 0 1 0 1.41Zm12.08-2.12a1 1 0 0 1 0 1.41l-.71.71a1 1 0 1 1-1.41-1.41l.71-.71a1 1 0 0 1 1.41 0ZM6.46 17.54a1 1 0 0 1 0 1.41l-.71.71A1 1 0 0 1 3.64 18.25l.71-.71a1 1 0 0 1 1.41 0Z"/></svg>
      </button>
      <div class="profile">
        <button class="avatar" @click="user ? showProfileMenu = !showProfileMenu : showAuth = true">
          <span>{{ avatarLabel }}</span>
        </button>
        <div v-if="showProfileMenu" class="menu">
          <div class="menu-item">{{ user?.name || user?.email }}</div>
          <button class="menu-item" @click="openAccountEdit(); showProfileMenu = false">Modifier le compte</button>
          <button class="menu-item" @click="logout">Se déconnecter</button>
        </div>
      </div>
    </div>
    <nav class="subnav">
      <button class="subnav-link" :class="{ active: tab === 'home' }" @click="goHome(); tab = 'home'">Accueil</button>
      <button class="subnav-link" :class="{ active: tab === 'games' }" @click="goHome(); tab = 'games'">Jeux</button>
    </nav>
  </div>
  <main class="container" v-if="currentPage === 'home'">
    <header class="header" v-show="tab === 'home'">
      <h1>Calculeur de calories</h1>
    </header>

    <section class="card" v-show="tab === 'home'">
      <h2>Vos informations</h2>
      <form class="grid advanced-form" @submit.prevent="calculate">
        <label class="field">
          <input type="number" min="0" step="1" v-model="age" placeholder="Entrez votre âge" required />
          <span class="label-text">Âge</span>
        </label>
        <div class="field">
          <span class="label-text">Sexe</span>
          <div class="segmented">
            <label class="seg-option">
              <input type="radio" value="homme" v-model="sex" />
              <span>Homme</span>
            </label>
            <label class="seg-option">
              <input type="radio" value="femme" v-model="sex" />
              <span>Femme</span>
            </label>
          </div>
        </div>
        <label class="field">
          <input type="number" min="0" step="0.1" v-model="height" placeholder="Ex: 178" required />
          <span class="label-text">Taille (cm)</span>
        </label>
        <label class="field">
          <input type="number" min="0" step="0.1" v-model="weight" placeholder="Ex: 72.5" required />
          <span class="label-text">Poids (kg)</span>
        </label>
        <label class="field">
          <input type="number" min="0" max="100" step="0.1" v-model="bodyFat" placeholder="Optionnel" />
          <span class="label-text">Masse grasse (%)</span>
        </label>
        <label class="field select-field">
          <UiSelect v-model="activity" :options="activityOptions" placeholder="Sélectionnez un niveau" />
          <span class="label-text">Niveau d’activité</span>
        </label>
        <div class="actions">
          <button type="submit" class="primary">Calculer</button>
        </div>
      </form>
    </section>

    <section v-show="hasCalculated && tab === 'home'" class="card">
      <h2>Résultats</h2>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Formule</th>
              <th>Maintien</th>
              <th>Perte 0,25 kg/sem</th>
              <th>Perte 0,5 kg/sem</th>
              <th>Perte 1 kg/sem</th>
              <th>Prise 0,25 kg/sem</th>
              <th>Prise 0,5 kg/sem</th>
              <th>Prise 1 kg/sem</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in results" :key="row.label">
              <td>{{ row.label }}</td>
              <td>{{ row.maintain }} kcal/j</td>
              <td>{{ row.lose025 }} kcal/j</td>
              <td>{{ row.lose05 }} kcal/j</td>
              <td>{{ row.lose1 }} kcal/j</td>
              <td>{{ row.gain025 }} kcal/j</td>
              <td>{{ row.gain05 }} kcal/j</td>
              <td>{{ row.gain1 }} kcal/j</td>
            </tr>
          </tbody>
        </table>
      </div>

      
    </section>

    <section class="card" v-show="tab === 'games'">
      <h2>Jeux</h2>
      <div class="games-list">
        <div class="game-item">
          <div class="game-title">Healthy Run</div>
          <div class="game-actions"><button class="primary" type="button" @click="goRun">Jouer</button></div>
        </div>
      </div>
    </section>
    <section v-show="hasCalculated && tab === 'home'" class="card">
      <h2>Plan nutritionnel</h2>
      <form class="grid" @submit.prevent>
        <label>
          Objectif
          <select v-model="goal">
            <option value="maintain">Maintien</option>
            <option value="lose025">Perte 0,25 kg/sem</option>
            <option value="lose05">Perte 0,5 kg/sem</option>
            <option value="lose1">Perte 1 kg/sem</option>
            <option value="gain025">Prise 0,25 kg/sem</option>
            <option value="gain05">Prise 0,5 kg/sem</option>
            <option value="gain1">Prise 1 kg/sem</option>
          </select>
        </label>
        <label>
          Répartition des macros
          <select v-model="macroPreset">
            <option value="equilibre">Équilibré</option>
            <option value="hyperproteine">Hyperprotéiné</option>
            <option value="pauvre_en_glucides">Pauvre en glucides</option>
          </select>
        </label>
      </form>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Calories</th>
              <th>Protéines</th>
              <th>Lipides</th>
              <th>Glucides</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td>{{ macros.calories }} kcal/j</td>
              <td>{{ macros.proteinGrams }} g/j</td>
              <td>{{ macros.fatGrams }} g/j</td>
              <td>{{ macros.carbsGrams }} g/j</td>
            </tr>
          </tbody>
        </table>
      </div>
      <div class="actions" style="margin-top:12px;">
        <button class="primary" type="button" @click="savePlan">Enregistrer le plan</button>
      </div>
      <div v-if="planSaveMsg" style="color:#a6a6a6; margin-top:8px;">{{ planSaveMsg }}</div>
    </section>
    <section class="card" v-show="tab === 'home'">
      <h2>Historique des plans</h2>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Date</th>
              <th>Objectif</th>
              <th>Préréglage</th>
              <th>Calories</th>
              <th>Prot</th>
              <th>Lip</th>
              <th>Glu</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(p,i) in savedPlans" :key="p.date">
              <td>{{ new Date(p.date).toLocaleString() }}</td>
              <td>{{ p.goal }}</td>
              <td>{{ p.preset }}</td>
              <td>{{ p.calories }}</td>
              <td>{{ p.proteinGrams }} g</td>
              <td>{{ p.fatGrams }} g</td>
              <td>{{ p.carbsGrams }} g</td>
              <td>
                <button type="button" @click="openPlanPreview(p)" class="primary" style="padding:6px 10px;">Charger</button>
                <button type="button" @click="deletePlanAt(i)" class="link" style="margin-left:8px;">Supprimer</button>
              </td>
            </tr>
            <tr v-if="savedPlans.length === 0">
              <td colspan="8" style="color: var(--muted);">Aucun plan enregistré</td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
    
    
    <div v-if="showAuth" class="modal">
      <div class="overlay" @click="showAuth = false"></div>
      <div class="modal-card">
        <h3>{{ authMode === 'login' ? 'Se connecter' : 'Créer un compte' }}</h3>
        <form class="modal-form" @submit.prevent="onAuthSubmit">
          <label>
            Email
            <input type="email" v-model="authEmail" required />
          </label>
          <label v-if="authMode === 'register'">
            Nom
            <input type="text" v-model="authName" />
          </label>
          <label>
            Mot de passe
            <input type="password" v-model="authPassword" required />
          </label>
          <div v-if="authError" style="color:#e74c3c; font-size:14px;">{{ authError }}</div>
          <div class="modal-actions">
            <button type="submit" class="primary" :disabled="authLoading">{{ authMode === 'login' ? 'Connexion' : 'Créer un compte' }}</button>
            <button type="button" class="link" @click="authMode = authMode === 'login' ? 'register' : 'login'">{{ authMode === 'login' ? 'Créer un compte' : 'J’ai déjà un compte' }}</button>
          </div>
        </form>
      </div>
    </div>
    <div v-if="showAccountEdit" class="modal">
      <div class="overlay" @click="closeAccountEdit"></div>
      <div class="modal-card">
        <h3>Modifier le compte</h3>
        <form class="modal-form" @submit.prevent="onAccountSave">
          <label>
            Nom
            <input type="text" v-model="editName" required />
          </label>
          <label>
            Email
            <input type="email" v-model="editEmail" required />
          </label>
          <label>
            Mot de passe actuel
            <input type="password" v-model="editCurrentPassword" placeholder="Requis si vous changez le mot de passe" />
          </label>
          <label>
            Nouveau mot de passe
            <input type="password" v-model="editNewPassword" placeholder="Optionnel" />
          </label>
          <label>
            Confirmer le nouveau mot de passe
            <input type="password" v-model="editConfirmPassword" placeholder="Optionnel" />
          </label>
          <div v-if="editError" style="color:#e74c3c; font-size:14px;">{{ editError }}</div>
          <div class="modal-actions">
            <button type="submit" class="primary" :disabled="editLoading">Enregistrer</button>
            <button type="button" class="link" @click="closeAccountEdit">Annuler</button>
          </div>
        </form>
      </div>
    </div>
    <div v-if="showPlanPreview" class="modal">
      <div class="overlay" @click="closePlanPreview"></div>
      <div class="modal-card plan-preview">
        <h3>Aperçu du plan</h3>
        <div class="preview-grid">
          <div class="preview-item"><span class="label">Date</span><span class="value">{{ new Date(previewPlan.date).toLocaleString() }}</span></div>
          <div class="preview-item"><span class="label">Objectif</span><span class="value">{{ previewPlan.goal }}</span></div>
          <div class="preview-item"><span class="label">Préréglage</span><span class="value">{{ previewPlan.preset }}</span></div>
          <div class="preview-item"><span class="label">Calories</span><span class="value">{{ previewPlan.calories }} kcal/j</span></div>
          <div class="preview-item"><span class="label">Protéines</span><span class="value">{{ previewPlan.proteinGrams }} g/j</span></div>
          <div class="preview-item"><span class="label">Lipides</span><span class="value">{{ previewPlan.fatGrams }} g/j</span></div>
          <div class="preview-item"><span class="label">Glucides</span><span class="value">{{ previewPlan.carbsGrams }} g/j</span></div>
          <div class="preview-item"><span class="label">Poids</span><span class="value">{{ previewPlan.weight }} kg</span></div>
        </div>
        <div class="modal-actions">
          <button type="button" class="link" @click="closePlanPreview">Fermer</button>
        </div>
      </div>
    </div>
  </main>
  <main class="container" v-else>
    <header class="header">
      <h1>Healthy Run – Le Parcours Santé</h1>
      <p>Attrape les aliments sains, évite le junk, deviens super fit.</p>
    </header>
    <section class="card">
      <h2>Contrôles</h2>
      <div class="grid">
        <div>← Aller à gauche</div>
        <div>→ Aller à droite</div>
      </div>
    </section>
    <section class="card">
      <h2>Jeu</h2>
      <div style="display:flex; gap:12px; align-items:center;">
        <button class="primary" type="button" @click="startRun">Start</button>
        <button class="link" type="button" @click="stopRun">Stop</button>
        <button class="link" type="button" @click="goHome">Retour</button>
        <div style="color: var(--muted);">Distance: {{ runDistance }} m • Santé: {{ runHealth }} • Vitesse: {{ runSpeed }}</div>
      </div>
      <div class="run-container">
        <div class="lane-grid"></div>
        <div class="runner-wrap" :style="{ left: (runAvatarLane*33.333)+'%' }">
          <div v-if="runAvatarSrc" :class="['runner','level-'+runLevel]" :style="{ backgroundImage: 'url(' + runAvatarSrc + ')' }"></div>
          <div v-else :class="['runner','level-'+runLevel]">{{ runEmoji }}</div>
        </div>
        <div v-for="it in runItems" :key="it.id" class="item" :class="it.healthy ? 'item-healthy' : 'item-junk'" :style="{ top: it.y+'px', left: (it.lane*33.333)+'%' }">
          <div class="item-icon">{{ it.icon }}</div>
        </div>
      </div>
      <div style="margin-top:10px; display:flex; gap:14px; color: var(--muted); align-items:center;">
        <div>Niveau: {{ runLevel }}</div>
        <div>Combo sain: {{ runCombo }}</div>
        <div>Meilleur score: {{ runBest }}</div>
        <div style="flex:1"></div>
      </div>
    </section>
  </main>
</template>

<style scoped>
.appbar { display: grid; grid-template-columns: 1fr auto 1fr; align-items: center; padding: 12px 20px; border-bottom: none; background: transparent; position: sticky; top: 0; z-index: 50; }
.brand { grid-column: 2; justify-self: center; font-family: 'Montserrat', system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Oxygen, Fira Sans, Droid Sans, Helvetica Neue, Arial, sans-serif; font-weight: 900; letter-spacing: .8px; text-transform: uppercase; color: var(--accent); font-size: 28px; text-shadow: 0 1px 0 rgba(0,0,0,0.25); }
.actions-bar { grid-column: 3; justify-self: end; display: flex; align-items: center; gap: 10px; }
.icon-btn { background: transparent; color: var(--accent); border: 1px solid rgba(255,204,0,0.45); border-radius: 999px; padding: 8px 10px; display: inline-flex; align-items: center; justify-content: center; transition: transform .18s ease, filter .18s ease, box-shadow .18s ease; }
.icon-btn:hover { transform: translateY(-2px); filter: brightness(1.05); box-shadow: 0 8px 20px rgba(255,204,0,0.18); }
.profile { position: relative; }
.avatar { background: var(--input-bg); color: var(--accent); border: 1px solid var(--input-border); width: 36px; height: 36px; border-radius: 999px; display: inline-flex; align-items: center; justify-content: center; font-weight: 700; }
.menu { position: absolute; right: 0; top: calc(100% + 8px); background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 10px; min-width: 200px; box-shadow: 0 10px 24px var(--shadow-color); overflow: hidden; }
.menu-item { display: block; width: 100%; text-align: left; padding: 10px 12px; color: var(--text); background: transparent; border: none; cursor: pointer; }
.menu-item:hover { background: rgba(255,204,0,0.06); }
.subnav { grid-column: 1 / -1; justify-self: center; display: flex; gap: 28px; padding: 8px 0 6px; }
.subnav-link { background: transparent; border: none; color: var(--muted); font-weight: 700; letter-spacing: .2px; padding: 6px 2px; cursor: pointer; }
.subnav-link:hover { color: var(--text); }
.subnav-link.active { color: var(--accent); }
.container { max-width: 980px; margin: 0 auto; padding: 28px; }
.header { display: flex; flex-direction: column; gap: 4px; align-items: center; justify-content: center; }
.header h1 { margin: 0; font-size: 22px; letter-spacing: .3px; color: var(--text); text-align: center; }
.card { background: var(--card-bg); backdrop-filter: blur(12px); border: 1px solid var(--card-border); border-radius: 16px; padding: 35px 40px; box-shadow: 0 20px 40px var(--shadow-color), 0 0 20px rgba(255,204,0,0.08); position: relative; animation: fadeUp .45s ease both; }
.card::before { content: ""; position: absolute; inset: 0; border-radius: 16px; pointer-events: none; box-shadow: 0 0 0 1px rgba(255,204,0,0.10) inset; }
.card + .card { margin-top: 24px; }
.card > h2 { font-size: 22px; font-weight: 600; letter-spacing: 0.5px; margin: 0 0 14px; }
.grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 18px; }
label { display: flex; flex-direction: column; gap: 8px; font-weight: 600; color: var(--text); }
.advanced-form { row-gap: 22px; }
.field { position: relative; display: flex; flex-direction: column; gap: 10px; animation: fieldFade .35s ease both; }
.advanced-form .field:nth-child(1) { animation-delay: .02s; }
.advanced-form .field:nth-child(2) { animation-delay: .06s; }
.advanced-form .field:nth-child(3) { animation-delay: .1s; }
.advanced-form .field:nth-child(4) { animation-delay: .14s; }
.advanced-form .field:nth-child(5) { animation-delay: .18s; }
.advanced-form .field:nth-child(6) { animation-delay: .22s; }
.label-text { position: absolute; top: -10px; left: 12px; font-size: 12px; color: var(--muted); letter-spacing: .2px; background: var(--card-bg); padding: 0 6px; border-radius: 6px; transition: top .25s cubic-bezier(.2,.7,.2,1), left .25s cubic-bezier(.2,.7,.2,1), transform .25s cubic-bezier(.2,.7,.2,1), font-size .25s ease, background-color .25s ease, color .25s ease, padding .25s ease; will-change: top, left, transform; }
.field:focus-within .label-text { color: var(--text); }
.field input::placeholder { opacity: 0; transition: opacity .18s ease; }
.field select::placeholder { opacity: 0; }
.field input:focus:placeholder-shown + .label-text {
  top: 50%; left: 16px; transform: translateY(-50%);
  font-size: 14px; background: transparent; color: var(--muted); padding: 0 8px;
}
.field select:focus + .label-text {
  top: -10px; left: 12px; transform: none;
  font-size: 12px; background: var(--card-bg); color: var(--muted); padding: 0 6px;
}
.field input:not(:placeholder-shown) + .label-text,
.field select + .label-text {
  top: -10px; left: 12px; transform: none; font-size: 12px; padding: 0 6px;
}
@keyframes fieldFade { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: translateY(0); } }
input, select { background: var(--input-bg); color: var(--text); border: 1px solid var(--input-border); border-radius: 12px; padding: 14px 16px; font-size: 14px; height: var(--input-h); box-shadow: 0 2px 6px rgba(0,0,0,0.06); transition: border-color .2s ease, box-shadow .2s ease, transform .2s ease; }
input, select { box-sizing: border-box; line-height: 1.25; }
input::placeholder, select::placeholder { color: var(--placeholder); }
input:focus, select:focus { outline: none; border-color: #f1c40f; box-shadow: 0 0 0 3px rgba(241,196,15,0.2); transform: translateY(-1px); }
.select-field select { appearance: auto; -webkit-appearance: auto; -moz-appearance: auto; padding-right: 16px; }
.select-field::after { display: none; }
select option { color: inherit; background: initial; }
.fieldset { border: 1px solid #30363d; border-radius: 10px; padding: 12px 14px; }
.inline { display: inline-flex; align-items: center; gap: 8px; font-weight: 500; color: var(--text); }
.inline input[type="radio"] { accent-color: #f1c40f; }
.segmented { display: inline-flex; gap: 8px; background: var(--input-bg); border: 1px solid var(--input-border); border-radius: 12px; padding: 6px; }
.seg-option { position: relative; display: inline-flex; }
.seg-option input { position: absolute; opacity: 0; pointer-events: none; }
.seg-option span { display: inline-block; padding: 8px 14px; border-radius: 10px; color: var(--text); transition: background-color .18s ease, color .18s ease, transform .18s ease; }
.seg-option:hover span { transform: translateY(-1px); }
.seg-option input:checked + span { background: rgba(255,204,0,0.18); color: var(--text); box-shadow: 0 0 0 2px rgba(255,204,0,0.35) inset; }
.actions { grid-column: 1 / -1; display: flex; justify-content: center; }
.primary { background: #FFCC00; color: #111; border: none; padding: 14px 32px; border-radius: 12px; cursor: pointer; font-weight: 700; letter-spacing: .2px; box-shadow: 0 5px 15px rgba(255,204,0,0.35); transition: transform .15s ease, box-shadow .15s ease, filter .15s ease; animation: pulseAccent 2.4s ease infinite; min-width: 260px; }
.primary:hover { transform: scale(1.03); box-shadow: 0 8px 20px rgba(255,204,0,0.45); filter: saturate(1.05) brightness(1.03); }
.primary:disabled { opacity: .6; cursor: not-allowed; box-shadow: none; }
 .games-list { display: grid; gap: 12px; }
 .game-item { display: flex; align-items: center; justify-content: space-between; background: var(--input-bg); border: 1px solid var(--input-border); border-radius: 12px; padding: 14px 16px; }
 .game-title { color: var(--text); font-weight: 600; }
 .table-wrap { overflow-x: auto; margin-top: 18px; border-radius: 12px; }
 table { width: 100%; border-collapse: separate; border-spacing: 0; }
 thead th { position: sticky; top: 0; z-index: 1; }
 th, td { border-bottom: 1px solid #1f2937; padding: 14px 16px; }
th { background: var(--thead-bg); color: var(--text); font-weight: 700; letter-spacing: .2px; }
 tbody tr { transition: background-color .18s ease; }
 tbody tr:nth-child(odd) { background: rgba(255,204,0,0.03); }
 tbody tr:hover { background: rgba(255,204,0,0.06); }
 td:not(:first-child) { text-align: right; font-variant-numeric: tabular-nums; }
.notes { margin-top: 16px; color: var(--muted); }
@media (max-width: 640px) { .grid { grid-template-columns: 1fr; } }

.modal { position: fixed; inset: 0; display: grid; place-items: center; z-index: 100; }
.overlay { position: absolute; inset: 0; background: rgba(0,0,0,0.6); backdrop-filter: blur(2px); }
.modal-card { position: relative; background: var(--card-bg); border: 1px solid var(--card-border); border-radius: 14px; padding: 18px; width: 92%; max-width: 420px; box-shadow: 0 20px 40px var(--shadow-color); animation: fadeUp .25s ease both; }
.modal-card h3 { margin: 0 0 10px 0; color: var(--text); }
.modal-form { display: grid; gap: 12px; }
.modal-actions { display: flex; gap: 10px; align-items: center; }
.link { background: transparent; color: var(--accent); border: none; padding: 10px; cursor: pointer; }
.form-message { margin-top: 8px; font-size: 14px; }
.form-message.error { color: var(--error); }
.form-message.success { color: var(--success); }
.plan-preview { background: radial-gradient(120% 120% at 10% 0%, rgba(255,204,0,0.10) 0%, rgba(17,22,30,1) 40%), var(--card-bg); border: 1px solid rgba(255,204,0,0.25); box-shadow: 0 25px 50px var(--shadow-color), 0 0 0 1px rgba(255,204,0,0.08) inset; }
.plan-preview h3 { font-weight: 700; letter-spacing: .3px; }
.preview-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; margin-top: 6px; }
.preview-item { display: flex; align-items: center; justify-content: space-between; padding: 10px 12px; border: 1px solid var(--card-border); border-radius: 10px; background: rgba(255,255,255,0.02); }
.preview-item .label { color: var(--muted); font-weight: 600; }
.preview-item .value { color: var(--text); font-weight: 700; }
.run-container { position: relative; height: 300px; margin-top: 12px; border-radius: 14px; overflow: hidden; background: linear-gradient(180deg, rgba(255,204,0,0.06), rgba(255,204,0,0.0)); box-shadow: inset 0 -30px 60px rgba(255,204,0,0.08); border: 1px solid #30363d; }
.lane-grid { position: absolute; inset: 0; display: grid; grid-template-columns: repeat(3, 1fr); background: linear-gradient(180deg, rgba(17,22,30,0.9), rgba(17,22,30,0.7)); }
.lane-grid::before { content:""; position:absolute; inset:0; background: repeating-linear-gradient(180deg, rgba(255,255,255,0.04) 0, rgba(255,255,255,0.04) 10px, transparent 10px, transparent 20px); mix-blend-mode: overlay; }
.lane-grid > :nth-child(1), .lane-grid > :nth-child(2) { border-right: 1px dashed #38414e; }
.runner-wrap { position: absolute; bottom: 14px; width: 33.333%; display: grid; place-items: center; transition: left .12s ease; }
.runner { width: 72px; height: 72px; border-radius: 999px; background: #0f141b center/cover no-repeat; border: 2px solid #30363d; display: grid; place-items: center; font-size: 30px; box-shadow: 0 10px 22px rgba(0,0,0,0.35), 0 0 0 6px rgba(255,204,0,0.06); }
.runner.level-1 { box-shadow: 0 0 0 4px rgba(255,255,255,0.06); }
.runner.level-2 { box-shadow: 0 0 0 6px rgba(46,204,113,0.15); border-color: rgba(46,204,113,0.5); }
.runner.level-3 { box-shadow: 0 0 0 8px rgba(52,152,219,0.18); border-color: rgba(52,152,219,0.6); }
.runner.level-4 { box-shadow: 0 0 0 10px rgba(241,196,15,0.22); border-color: rgba(241,196,15,0.8); }
.item { position: absolute; width: 33.333%; display: grid; place-items: center; }
.item-icon { font-size: 26px; padding: 8px 10px; border-radius: 10px; backdrop-filter: blur(2px); box-shadow: 0 6px 14px rgba(0,0,0,0.25); }
.item-healthy .item-icon { background: rgba(46,204,113,0.18); border: 1px solid rgba(46,204,113,0.45); }
.item-junk .item-icon { background: rgba(231,76,60,0.18); border: 1px solid rgba(231,76,60,0.45); }
</style>
