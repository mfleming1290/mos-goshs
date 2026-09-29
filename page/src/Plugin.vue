<template>
  <div class="goshs-page">
    <div class="header-row">
      <div>
        <h2>GoSHS</h2>
        <p class="muted">A lightweight file server for binaries and other MOS files.</p>
      </div>
      <div class="status-pill" :class="{ running }">
        <span class="dot"></span>
        {{ running ? 'Running' : 'Stopped' }}
      </div>
    </div>

    <div class="grid">
      <section class="card">
        <h3>Runtime</h3>
        <div class="kv"><span>Installed</span><strong>{{ currentVersion || 'Not installed' }}</strong></div>
        <div class="kv"><span>Latest</span><strong>{{ latestVersion || 'Unknown' }}</strong></div>

        <div class="actions">
          <button class="primary" :disabled="busy" @click="installOrUpdate">{{ installLabel }}</button>
          <button :disabled="busy || !currentVersion" @click="restartServer">Restart</button>
        </div>
      </section>

      <section class="card">
        <h3>Service</h3>
        <div class="actions">
          <button class="primary" :disabled="busy || running || !currentVersion" @click="startServer">Start</button>
          <button :disabled="busy || !running" @click="stopServer">Stop</button>
          <button :disabled="!running" @click="openWebUi">Open GoSHS</button>
        </div>
        <p class="hint">The full GoSHS CLI is installed as <code>/usr/bin/goshs</code>.</p>
      </section>
    </div>

    <section class="card settings-card">
      <h3>Server settings</h3>
      <div class="form-grid">
        <label>
          <span>Directory</span>
          <input v-model.trim="settings.webroot" type="text" placeholder="/mnt/user/binaries" />
        </label>
        <label>
          <span>Listen IP</span>
          <input v-model.trim="settings.listen_ip" type="text" placeholder="0.0.0.0" />
        </label>
        <label>
          <span>Port</span>
          <input v-model.number="settings.port" type="number" min="1" max="65535" />
        </label>
        <label>
          <span>WebDAV port</span>
          <input v-model.number="settings.webdav_port" type="number" min="1" max="65535" :disabled="!settings.webdav" />
        </label>
      </div>

      <div class="toggles">
        <label><input v-model="settings.auto_start" type="checkbox" /> Auto start</label>
        <label><input v-model="settings.read_only" type="checkbox" /> Read only</label>
        <label><input v-model="settings.no_chat" type="checkbox" /> Disable chat</label>
        <label><input v-model="settings.webdav" type="checkbox" /> Enable WebDAV</label>
      </div>

      <div class="actions">
        <button class="primary" :disabled="busy || !validSettings" @click="saveSettings(true)">Save & Apply</button>
        <button :disabled="busy" @click="loadAll">Reload</button>
      </div>
    </section>

    <p v-if="message" class="message" :class="messageType">{{ message }}</p>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue';

const PLUGIN_NAME = 'goshs';
const busy = ref(false);
const running = ref(false);
const currentVersion = ref('');
const latestVersion = ref('');
const updateAvailable = ref(false);
const message = ref('');
const messageType = ref('info');
let pollTimer = null;

const settings = reactive({
  auto_start: true,
  webroot: '/mnt/user/binaries',
  listen_ip: '0.0.0.0',
  port: 8000,
  read_only: true,
  no_chat: true,
  webdav: false,
  webdav_port: 8001,
});

const getAuthHeaders = () => {
  const token = localStorage.getItem('token') || localStorage.getItem('authToken') || '';
  return token ? { Authorization: `Bearer ${token}` } : {};
};

const validPort = (value) => Number.isInteger(Number(value)) && Number(value) >= 1 && Number(value) <= 65535;
const validSettings = computed(() =>
  settings.webroot.length > 0 &&
  settings.listen_ip.length > 0 &&
  validPort(settings.port) &&
  (!settings.webdav || validPort(settings.webdav_port))
);

const installLabel = computed(() => {
  if (!currentVersion.value) return busy.value ? 'Installing…' : 'Install GoSHS';
  if (updateAvailable.value) return busy.value ? 'Updating…' : `Update to ${latestVersion.value}`;
  return 'Reinstall latest';
});

const pluginQuery = async (args, timeout = 30) => {
  const res = await fetch('/api/v1/mos/plugins/query', {
    method: 'POST',
    headers: {
      ...getAuthHeaders(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      command: PLUGIN_NAME,
      args,
      timeout,
      parse_json: false,
    }),
  });
  if (!res.ok) {
    let detail = '';
    try { detail = JSON.stringify(await res.json()); } catch (_) {}
    throw new Error(`Plugin command failed (${res.status})${detail ? `: ${detail}` : ''}`);
  }
  return res.json();
};

const commandOutput = (data) => {
  if (data == null) return '';
  if (typeof data === 'string') return data;
  return data.output ?? data.stdout ?? data.result ?? '';
};

const parseCommandJson = (data) => {
  if (data && typeof data === 'object') {
    if (data.running !== undefined || data.current !== undefined) return data;
    const out = commandOutput(data);
    if (typeof out === 'object') return out;
    if (typeof out === 'string' && out.trim()) return JSON.parse(out);
  }
  if (typeof data === 'string' && data.trim()) return JSON.parse(data);
  return {};
};

const loadSettings = async () => {
  const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, { headers: getAuthHeaders() });
  if (!res.ok) throw new Error(`Settings request failed (${res.status})`);
  const data = await res.json();
  for (const key of Object.keys(settings)) {
    if (data[key] !== undefined && data[key] !== null) settings[key] = data[key];
  }
};

const persistSettings = async () => {
  const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
    method: 'POST',
    headers: {
      ...getAuthHeaders(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      ...settings,
      port: Number(settings.port),
      webdav_port: Number(settings.webdav_port),
      version: currentVersion.value || '',
    }),
  });
  if (!res.ok) throw new Error(`Saving settings failed (${res.status})`);
};

const saveSettings = async (applyAfterSave = false) => {
  if (!validSettings.value) return;
  busy.value = true;
  message.value = '';
  try {
    await persistSettings();
    if (applyAfterSave && running.value) await pluginQuery(['restart'], 20);
    messageType.value = 'success';
    message.value = applyAfterSave && running.value ? 'Settings saved and GoSHS restarted.' : 'Settings saved.';
    await refreshStatus();
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  } finally {
    busy.value = false;
  }
};

const refreshStatus = async () => {
  try {
    const data = parseCommandJson(await pluginQuery(['status'], 5));
    running.value = Boolean(data.running);
  } catch (_) {
    running.value = false;
  }
};

const refreshVersion = async () => {
  try {
    const data = parseCommandJson(await pluginQuery(['check_version'], 15));
    currentVersion.value = data.current && data.current !== 'not installed' ? data.current : '';
    latestVersion.value = data.latest || '';
    updateAvailable.value = Boolean(data.update_available);
  } catch (_) {
    latestVersion.value = '';
    updateAvailable.value = false;
  }
};

const installOrUpdate = async () => {
  busy.value = true;
  message.value = '';
  try {
    await pluginQuery(['install_binary'], 120);
    await Promise.all([refreshVersion(), refreshStatus()]);
    messageType.value = 'success';
    message.value = `GoSHS ${currentVersion.value || 'runtime'} installed.`;
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  } finally {
    busy.value = false;
  }
};

const startServer = async () => {
  busy.value = true;
  message.value = '';
  try {
    await persistSettings();
    await pluginQuery(['start'], 20);
    await refreshStatus();
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  } finally {
    busy.value = false;
  }
};

const stopServer = async () => {
  busy.value = true;
  message.value = '';
  try {
    await pluginQuery(['stop'], 20);
    await refreshStatus();
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  } finally {
    busy.value = false;
  }
};

const restartServer = async () => {
  busy.value = true;
  message.value = '';
  try {
    await persistSettings();
    await pluginQuery(['restart'], 20);
    await refreshStatus();
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  } finally {
    busy.value = false;
  }
};

const openWebUi = () => {
  const host = window.location.hostname;
  window.open(`http://${host}:${settings.port}/`, '_blank', 'noopener,noreferrer');
};

const loadAll = async () => {
  message.value = '';
  try {
    await loadSettings();
    await Promise.all([refreshStatus(), refreshVersion()]);
  } catch (error) {
    messageType.value = 'error';
    message.value = error.message;
  }
};

onMounted(async () => {
  await loadAll();
  pollTimer = setInterval(refreshStatus, 5000);
});

onBeforeUnmount(() => {
  if (pollTimer) clearInterval(pollTimer);
});
</script>

<style scoped>
.goshs-page { padding: 24px; max-width: 1050px; }
.header-row { display: flex; justify-content: space-between; gap: 20px; align-items: flex-start; margin-bottom: 18px; }
h2, h3 { margin-top: 0; }
.muted, .hint { opacity: .72; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 16px; margin-bottom: 16px; }
.card { border: 1px solid rgba(128,128,128,.35); border-radius: 12px; padding: 20px; background: rgba(128,128,128,.06); }
.settings-card { margin-top: 4px; }
.status-pill { display: inline-flex; align-items: center; gap: 8px; border: 1px solid rgba(128,128,128,.35); border-radius: 999px; padding: 7px 12px; }
.dot { width: 9px; height: 9px; border-radius: 50%; background: #888; }
.status-pill.running .dot { background: #3fb950; }
.kv { display: flex; justify-content: space-between; gap: 16px; padding: 8px 0; border-bottom: 1px solid rgba(128,128,128,.18); }
.actions { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 16px; }
button { padding: 9px 14px; border-radius: 8px; border: 1px solid rgba(128,128,128,.5); cursor: pointer; }
button.primary { font-weight: 700; }
button:disabled { opacity: .5; cursor: not-allowed; }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 14px; }
label span { display: block; font-weight: 600; margin-bottom: 7px; }
input[type="text"], input[type="number"] { width: 100%; box-sizing: border-box; padding: 9px 11px; border-radius: 8px; border: 1px solid rgba(128,128,128,.45); background: transparent; color: inherit; }
.toggles { display: flex; flex-wrap: wrap; gap: 18px; margin-top: 18px; }
.toggles label { display: inline-flex; align-items: center; gap: 7px; }
.message { margin-top: 16px; padding: 11px 13px; border-radius: 8px; }
.message.success { background: rgba(46,160,67,.15); }
.message.error { background: rgba(218,54,51,.15); }
.message.info { background: rgba(56,139,253,.12); }
code { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; }
</style>