<script setup>
import { ref } from 'vue'

const props = defineProps({
    stremioAPIBase: {
        type: String,
        required: true
    },
})

const authKey = ref('')
const email = ref('')
const password = ref('')
const loginButtonText = ref('Login')
const emits = defineEmits(['auth-key', 'auth-success'])

async function login() {
    const cleanAuthKey = authKey.value.replaceAll('"', '').trim()
    const hasCredentials = email.value.trim() && password.value.trim()
    const hasAuthKey = cleanAuthKey.length > 0
    
    if (!hasCredentials && !hasAuthKey) {
        alert('Please provide either email/password or paste an AuthKey')
        return
    }
    
    try {
        loginButtonText.value = 'Logging in...'
        
        if (hasCredentials) {
            // Use username/password login
            fetch(`${props.stremioAPIBase}login`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    authKey: null,
                    email: email.value,
                    password: password.value,
                })
            }).then((resp) => {
                resp.json().then((data) => {
                    console.log("Auth data:", data)
                    if (data.result?.authKey) {
                        authKey.value = data.result.authKey
                        loginButtonText.value = 'Logged in'
                        emitAuthKey()
                        emits('auth-success', authKey.value)
                    } else {
                        throw new Error('Invalid credentials')
                    }
                })
            }).catch((error) => {
                throw error
            })
        } else if (hasAuthKey) {
            // Use direct authkey login - validate by making a test API call
            fetch(`${props.stremioAPIBase}addonCollectionGet`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    type: 'AddonCollectionGet',
                    authKey: cleanAuthKey,
                    update: false,
                })
            }).then((resp) => {
                resp.json().then((data) => {
                    console.log("AuthKey validation:", data)
                    if (data.result !== null && data.result !== undefined) {
                        loginButtonText.value = 'Logged in'
                        authKey.value = cleanAuthKey
                        emitAuthKey()
                        emits('auth-success', cleanAuthKey)
                    } else {
                        throw new Error('Invalid AuthKey')
                    }
                })
            }).catch((error) => {
                throw error
            })
        }
    } catch (err) {
        console.error(err);
        alert('Login failed: ' + (err.message || 'Unknown error'));
        loginButtonText.value = 'Login'
    }
}

function emitAuthKey() {
    const cleanAuthKey = authKey.value.replaceAll('"', '').trim()
    emits('auth-key', cleanAuthKey)
}

</script>

<template>
  <div class="auth theme-dark">
    <div class="field-row login-row">
      <input class="input" type="text" v-model="email" placeholder="Stremio E-mail" autocomplete="username">
      <input class="input" type="password" v-model="password" placeholder="Stremio Password" autocomplete="current-password">
      <button class="btn accent" @click="login" :disabled="loginButtonText === 'Logging in...'">{{ loginButtonText }}</button>
    </div>
    <div class="divider"><span>OR</span></div>
    <div class="field-row authkey-row">
      <input class="input" type="password" v-model="authKey" v-on:input="emitAuthKey" placeholder="Paste Stremio AuthKey here...">
    </div>
  </div>
</template>

<style scoped>
.auth {
  width: 100%;
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
.auth.theme-dark {
  --panel: #252526;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --control-height: 42px;
}
.field-row {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
  align-items: center;
}
@media (min-width: 640px) {
  .login-row { grid-template-columns: 2fr 2fr auto; }
}
.input {
  width: 100%;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid var(--border);
  background: #1e1e1e;
  color: var(--text);
  height: var(--control-height);
  box-sizing: border-box;
}
.divider {
  display: grid;
  place-items: center;
  color: var(--muted);
  font-weight: 700;
  letter-spacing: 1px;
  padding: 8px 0;
}
.btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 0 16px;
  border-radius: 8px;
  border: 1px solid var(--border);
  background: #2d2d30;
  color: var(--text);
  cursor: pointer;
  height: var(--control-height);
}
.login-row .btn { width: 100%; }
@media (min-width: 640px) { .login-row .btn { width: auto; justify-self: start; } }
.btn.accent { background: linear-gradient(180deg, var(--accent), var(--accent-2)); border-color: transparent; color: #fff; }
.auth-hint {
  font-size: 12px;
  color: var(--muted);
  margin: 8px 0 0 0;
  text-align: center;
}
</style>
