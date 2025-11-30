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
  </div>
  <main class="container">
    <header class="header">
      <h1>Calculeur de calories</h1>
    </header>

    <section class="card">
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
</style>
