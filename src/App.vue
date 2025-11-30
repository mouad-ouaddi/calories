<script setup>
import { ref, computed, onMounted } from 'vue'

const age = ref('')
const sex = ref('homme')
const height = ref('')
const weight = ref('')
const bodyFat = ref('')
const activity = ref('1.55')
const hasCalculated = ref(false)

const theme = ref('dark')
const user = ref(null)
const showAuth = ref(false)
const authMode = ref('login')
const authEmail = ref('')
const authPassword = ref('')
const authName = ref('')
const showProfileMenu = ref(false)

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
const onAuthSubmit = () => {
  const email = authEmail.value.trim()
  const pass = authPassword.value.trim()
  const name = authName.value.trim()
  if (authMode.value === 'register') {
    const u = { email, name }
    saveUser(u)
    showAuth.value = false
  } else {
    const raw = localStorage.getItem('app_user')
    if (raw) {
      try { user.value = JSON.parse(raw) } catch {}
      showAuth.value = false
    } else {
      authMode.value = 'register'
    }
  }
}

onMounted(() => {
  loadUser()
  const stored = localStorage.getItem('theme')
  theme.value = stored === 'light' ? 'light' : 'dark'
  applyTheme(theme.value)
})
</script>

<template>
  <div class="appbar">
    <div class="brand">Calories</div>
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
          <button class="menu-item" @click="logout">Se déconnecter</button>
        </div>
      </div>
    </div>
  </div>
  <main class="container">
    <header class="header">
      <h1>Calculateur de Calories</h1>
      <p>Calculez votre apport calorique quotidien en utilisant des formules reconnues.</p>
    </header>

    <section class="card">
      <h2>Vos informations</h2>
      <form class="grid" @submit.prevent="calculate">
        <label>
          Âge
          <input type="number" min="0" step="1" v-model="age" placeholder="années" required />
        </label>
        <fieldset class="fieldset">
          <legend>Sexe</legend>
          <label class="inline">
            <input type="radio" value="homme" v-model="sex" /> Homme
          </label>
          <label class="inline">
            <input type="radio" value="femme" v-model="sex" /> Femme
          </label>
        </fieldset>
        <label>
          Taille (cm)
          <input type="number" min="0" step="0.1" v-model="height" placeholder="cm" required />
        </label>
        <label>
          Poids (kg)
          <input type="number" min="0" step="0.1" v-model="weight" placeholder="kg" required />
        </label>
        <label>
          Masse grasse (%)
          <input type="number" min="0" max="100" step="0.1" v-model="bodyFat" placeholder="optionnel" />
        </label>
        <label>
          Niveau d’activité
          <select v-model="activity">
            <option value="1.2">Sédentaire (peu ou pas d’exercice)</option>
            <option value="1.375">Léger (15–30 min / jour)</option>
            <option value="1.55">Modéré (30–60 min / jour)</option>
            <option value="1.725">Intense (60–120 min / jour)</option>
            <option value="1.9">Très intense (2+ heures / jour)</option>
          </select>
        </label>
        <div class="actions">
          <button type="submit" class="primary">Calculer</button>
        </div>
      </form>
    </section>

    <section v-if="hasCalculated" class="card">
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

      <div class="notes">
        <p>
          Les formules utilisées: Mifflin-St Jeor, Harris-Benedict révisée, et Katch-McArdle
          (nécessite la masse grasse). Elles estiment le BMR, puis le TDEE est obtenu via un
          facteur d’activité.
        </p>
        <p>
          1 lb (~0,45 kg) ≈ 3 500 kcal; 1 kg ≈ 7 700 kcal. Un déficit excédant 1 000 kcal/j
          n’est généralement pas conseillé. Une perte trop rapide peut réduire la masse
          musculaire et donc le BMR.
        </p>
        <p>
          Associez ces chiffres à une alimentation équilibrée et une activité régulière.
        </p>
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
          <div class="modal-actions">
            <button type="submit" class="primary">{{ authMode === 'login' ? 'Connexion' : 'Créer un compte' }}</button>
            <button type="button" class="link" @click="authMode = authMode === 'login' ? 'register' : 'login'">{{ authMode === 'login' ? 'Créer un compte' : 'J’ai déjà un compte' }}</button>
          </div>
        </form>
      </div>
    </div>
  </main>
</template>

<style scoped>
.appbar { display: flex; align-items: center; justify-content: space-between; padding: 12px 20px; border-bottom: 1px solid #1f2937; background: #11161e; position: sticky; top: 0; z-index: 50; }
.brand { font-weight: 800; letter-spacing: .4px; color: var(--text); }
.actions-bar { display: flex; align-items: center; gap: 10px; }
.icon-btn { background: transparent; color: var(--accent); border: 1px solid rgba(255,204,0,0.45); border-radius: 999px; padding: 8px 10px; display: inline-flex; align-items: center; justify-content: center; transition: transform .18s ease, filter .18s ease, box-shadow .18s ease; }
.icon-btn:hover { transform: translateY(-2px); filter: brightness(1.05); box-shadow: 0 8px 20px rgba(255,204,0,0.18); }
.profile { position: relative; }
.avatar { background: #0f141b; color: var(--accent); border: 1px solid rgba(255,204,0,0.45); width: 36px; height: 36px; border-radius: 999px; display: inline-flex; align-items: center; justify-content: center; font-weight: 700; }
.menu { position: absolute; right: 0; top: calc(100% + 8px); background: #0f141b; border: 1px solid #30363d; border-radius: 10px; min-width: 200px; box-shadow: 0 10px 24px rgba(0,0,0,0.35); overflow: hidden; }
.menu-item { display: block; width: 100%; text-align: left; padding: 10px 12px; color: var(--text); background: transparent; border: none; cursor: pointer; }
.menu-item:hover { background: rgba(255,204,0,0.06); }
.container { max-width: 980px; margin: 0 auto; padding: 28px; }
.header { display: flex; flex-direction: column; gap: 6px; }
.header h1 { margin: 0; font-size: 32px; letter-spacing: .2px; color: var(--text); }
.header p { margin: 0; color: var(--muted); }
.card { background: rgba(30,30,30,0.75); backdrop-filter: blur(12px); border: 1px solid rgba(255,255,255,0.07); border-radius: 16px; padding: 35px 40px; box-shadow: 0 20px 40px rgba(0,0,0,0.35), 0 0 20px rgba(255,204,0,0.08); position: relative; animation: fadeUp .45s ease both; }
.card::before { content: ""; position: absolute; inset: 0; border-radius: 16px; pointer-events: none; box-shadow: 0 0 0 1px rgba(255,204,0,0.10) inset; }
.card > h2 { font-size: 22px; font-weight: 600; letter-spacing: 0.5px; margin: 0 0 14px; }
.grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 18px; }
label { display: flex; flex-direction: column; gap: 8px; font-weight: 600; color: var(--text); }
input, select { background: #111; color: var(--text); border: 1px solid #333; border-radius: 10px; padding: 12px 14px; font-size: 14px; transition: border-color .2s ease, box-shadow .2s ease, transform .2s ease; }
input::placeholder { color: #777; }
input:focus, select:focus { outline: none; border-color: #f1c40f; box-shadow: 0 0 0 3px rgba(241,196,15,0.2); transform: translateY(-1px); }
.fieldset { border: 1px solid #30363d; border-radius: 10px; padding: 12px 14px; }
.inline { display: inline-flex; align-items: center; gap: 8px; font-weight: 500; color: var(--text); }
.inline input[type="radio"] { accent-color: #f1c40f; }
.actions { grid-column: 1 / -1; display: flex; justify-content: flex-end; }
.primary { background: #f1c40f; color: var(--accent-dark); border: none; padding: 12px 26px; border-radius: 12px; cursor: pointer; font-weight: 700; letter-spacing: .2px; box-shadow: 0 5px 15px rgba(241,196,15,0.35); transition: transform .15s ease, box-shadow .15s ease, filter .15s ease; animation: pulseAccent 2.4s ease infinite; }
.primary:hover { transform: scale(1.03); box-shadow: 0 8px 20px rgba(241,196,15,0.45); filter: brightness(1.05); }
.table-wrap { overflow-x: auto; margin-top: 18px; }
table { width: 100%; border-collapse: collapse; }
th, td { border-bottom: 1px solid #1f2937; padding: 12px; text-align: left; }
th { background: #121822; color: var(--muted); }
tbody tr { transition: background-color .18s ease; }
tbody tr:hover { background: rgba(255,204,0,0.06); }
.notes { margin-top: 16px; color: var(--muted); }
@media (max-width: 640px) { .grid { grid-template-columns: 1fr; } }

.modal { position: fixed; inset: 0; display: grid; place-items: center; z-index: 100; }
.overlay { position: absolute; inset: 0; background: rgba(0,0,0,0.6); backdrop-filter: blur(2px); }
.modal-card { position: relative; background: #11161e; border: 1px solid #30363d; border-radius: 14px; padding: 18px; width: 92%; max-width: 420px; box-shadow: 0 20px 40px rgba(0,0,0,0.35); animation: fadeUp .25s ease both; }
.modal-card h3 { margin: 0 0 10px 0; color: var(--text); }
.modal-form { display: grid; gap: 12px; }
.modal-actions { display: flex; gap: 10px; align-items: center; }
.link { background: transparent; color: var(--accent); border: none; padding: 10px; cursor: pointer; }
</style>
