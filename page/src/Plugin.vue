<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_rclone.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Status Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cloud-sync</v-icon>
          <span>{{ $t('plugin_rclone.status') }}</span>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_rclone.version') }} {{ currentVersion || $t('plugin_rclone.not_installed') }}
              </span>
            </v-col>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_rclone.latest_version') }} {{ latestVersion || '-' }}
              </span>
            </v-col>
            <v-col v-if="webuiEnabled" cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_rclone.status') }}: {{ running ? $t('plugin_rclone.running') : $t('plugin_rclone.not_running') }}
              </span>
            </v-col>
            <v-col cols="12" md="auto" class="d-flex ga-2 flex-nowrap flex-md-row">
              <v-btn size="small" variant="tonal" color="secondary" @click="checkForUpdates" :loading="checkingUpdates">
                {{ $t('plugin_rclone.check_updates') }}
              </v-btn>
              <v-btn
                v-if="!currentVersion"
                size="small" variant="tonal" color="primary"
                @click="doInstall" :loading="installing"
              >
                {{ $t('plugin_rclone.install') }}
              </v-btn>
              <v-btn
                v-else-if="currentVersion && updateAvailable"
                size="small" variant="tonal" color="primary"
                @click="doUpdate" :loading="updating"
              >
                {{ $t('plugin_rclone.update') }}
              </v-btn>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Settings Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cog</v-icon>
          <span>{{ $t('plugin_rclone.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch
            v-if="webuiEnabled"
            v-model="settings.auto_start"
            :label="$t('plugin_rclone.auto_start')"
            inset color="green" hide-details
          />
          <v-text-field
            v-model="settings.webui_port"
            :label="$t('plugin_rclone.webui_port')"
            :hint="!webuiEnabled ? $t('plugin_rclone.default_port_hint', { port: 5572 }) : ''"
            :persistent-hint="!webuiEnabled"
            class="mt-4"
            type="number"
            style="max-width: 200px"
          />
          <v-btn color="primary" class="mt-4" @click="saveSettings" :loading="saving">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_rclone.save_settings') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Start/Stop Card -->
      <v-card v-if="currentVersion && webuiEnabled" class="mb-4 pa-0">
        <v-card-text class="d-flex align-center ga-2">
          <v-btn color="primary" rounded :loading="starting" @click="startDaemon">
            <v-icon start>mdi-play</v-icon>
            {{ $t('plugin_rclone.start') }}
          </v-btn>
          <v-btn color="error" rounded variant="outlined" :loading="stopping" @click="stopDaemon">
            <v-icon start>mdi-stop</v-icon>
            {{ $t('plugin_rclone.stop') }}
          </v-btn>
          <v-btn
            v-if="running"
            color="secondary" rounded variant="tonal"
            :href="webuiUrl" target="_blank"
          >
            <v-icon start>mdi-open-in-new</v-icon>
            {{ $t('plugin_rclone.open_webui') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Terminal Section -->
      <v-container v-if="currentVersion" fluid class="pt-2 pr-0 pl-0 pb-2">
        <div class="d-flex align-center ga-3 mb-4">
          <div style="width: 4px; height: 32px; border-radius: 2px; background: rgb(var(--v-theme-primary))"></div>
          <h2 class="font-weight-medium ma-0" style="font-weight: 600; line-height: 1.1">{{ $t('plugin_rclone.terminal') }}</h2>
          <v-spacer></v-spacer>
          <v-btn
            v-if="!terminalActive"
            text class="d-flex align-center" density="compact"
            @click="openTerminal"
          >
            <v-icon small class="mr-1">mdi-console</v-icon>
            {{ $t('plugin_rclone.open_terminal') }}
          </v-btn>
          <v-btn
            v-else
            text class="d-flex align-center" density="compact" color="error"
            @click="closeTerminal"
          >
            <v-icon small class="mr-1">mdi-close</v-icon>
            {{ $t('plugin_rclone.close_terminal') }}
          </v-btn>
        </div>
        <p v-if="!terminalActive" class="text-medium-emphasis ml-5">
          {{ $t('plugin_rclone.terminal_hint') }}
        </p>
        <div v-else>
          <div ref="terminalContainer" style="width: 100%; height: 420px; padding: 8px; background: #000000; border-radius: 8px"></div>
        </div>
      </v-container>
    </div>

    <!-- Overlay -->
    <v-overlay :model-value="overlay" class="align-center justify-center">
      <v-progress-circular color="onPrimary" size="64" indeterminate />
    </v-overlay>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, nextTick } from 'vue';
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';
import { ClipboardAddon } from '@xterm/addon-clipboard';
import { io } from 'socket.io-client';
import '@xterm/xterm/css/xterm.css';

const PLUGIN_NAME = 'rclone';

const loading = ref(true);
const saving = ref(false);
const overlay = ref(false);
const starting = ref(false);
const stopping = ref(false);
const checkingUpdates = ref(false);
const updating = ref(false);
const installing = ref(false);
const statusInterval = ref(null);

const running = ref(false);
const currentVersion = ref('');
const latestVersion = ref('');
const updateAvailable = ref(false);

const settings = reactive({
  auto_start: false,
  webui_port: '5572',
  version: ''
});

const terminalContainer = ref(null);
const terminalActive = ref(false);
let term = null;
let fitAddon = null;
let clipboardAddon = null;
let socket = null;
let terminalSessionId = null;
let resizeHandler = null;

const webuiEnabled = computed(() => {
  const port = parseInt(settings.webui_port, 10);
  return port > 0;
});

const webuiUrl = computed(() => {
  const port = settings.webui_port || '5572';
  return `${window.location.protocol}//${window.location.hostname}:${port}`;
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const doInstall = async () => {
  installing.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Install failed');
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to install:', e);
    alert('Failed to install rclone. Check logs for details.');
  } finally {
    installing.value = false;
  }
};

const doUpdate = async () => {
  updating.value = true;
  try {
    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'rclone',
          args: ['stop'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Update failed');
    }

    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'rclone',
          args: ['start'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to update:', e);
    alert('Failed to update rclone. Check logs for details.');
  } finally {
    updating.value = false;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.auto_start !== undefined) {
        settings.auto_start = data.auto_start;
      }
      if (data.webui_port !== undefined) {
        settings.webui_port = data.webui_port;
      }
      if (data.version) {
        currentVersion.value = data.version;
      }
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const checkStatus = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['status'],
        timeout: 5,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        running.value = data.output.running === true;
      }
    }
  } catch (e) {
    running.value = false;
  }
};

const checkForUpdates = async () => {
  checkingUpdates.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['check_version'],
        timeout: 15,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        latestVersion.value = data.output.latest || '';
        updateAvailable.value = currentVersion.value && data.output.update_available === true;
      }
    }
  } catch (e) {
    console.error('Failed to check updates:', e);
  } finally {
    checkingUpdates.value = false;
  }
};

const startDaemon = async () => {
  starting.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['start'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to start:', e);
  } finally {
    starting.value = false;
  }
};

const stopDaemon = async () => {
  stopping.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'rclone',
        args: ['stop'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to stop:', e);
  } finally {
    stopping.value = false;
  }
};

const saveSettings = async () => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        auto_start: settings.auto_start,
        webui_port: settings.webui_port,
        version: currentVersion.value,
      }),
    });
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const openTerminal = async () => {
  terminalActive.value = true;
  await nextTick();

  try {
    const res = await fetch('/api/v1/terminal/create', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: '/usr/bin/rcloneorig',
        args: ['config', '--config', '/boot/optional/plugins/rclone/rclone.conf'],
      }),
    });

    if (!res.ok) {
      throw new Error('Failed to create terminal session');
    }

    const session = await res.json();
    terminalSessionId = session.sessionId;

    fitAddon = new FitAddon();
    clipboardAddon = new ClipboardAddon();

    term = new Terminal({ cursorBlink: true, fontFamily: 'monospace', fontSize: 14 });
    term.loadAddon(fitAddon);
    term.loadAddon(clipboardAddon);
    term.open(terminalContainer.value);
    term.focus();
    fitAddon.fit();

    term.onSelectionChange(() => {
      if (!term.hasSelection()) return;
      const selected = term.getSelection();

      if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard.writeText(selected);
      } else {
        const textarea = document.createElement('textarea');
        textarea.value = selected;
        textarea.style.cssText = 'position:fixed;opacity:0';
        document.body.appendChild(textarea);
        textarea.focus();
        textarea.select();
        try {
          document.execCommand('copy');
        } catch (e) {}
        document.body.removeChild(textarea);
      }
    });

    socket = io('/terminal', { path: '/api/v1/socket.io/' });

    socket.on('connect', () => {
      socket.emit('join-session', {
        sessionId: terminalSessionId,
        token: localStorage.getItem('authToken'),
      });
    });

    socket.on('session-joined', () => {
      fitAddon.fit();
      socket.emit('terminal-resize', { cols: term.cols, rows: term.rows });
    });

    socket.on('terminal-output', (data) => {
      term.write(data);
    });

    term.onData((data) => {
      socket.emit('terminal-input', data);
    });

    term.onResize(({ cols, rows }) => {
      socket.emit('terminal-resize', { cols, rows });
    });

    resizeHandler = () => {
      if (fitAddon) fitAddon.fit();
    };
    window.addEventListener('resize', resizeHandler);

    socket.on('disconnect', () => {
      if (term) term.write('\r\nConnection closed.\r\n');
    });
  } catch (e) {
    console.error('Failed to open terminal:', e);
    terminalActive.value = false;
  }
};

const closeTerminal = () => {
  if (resizeHandler) {
    window.removeEventListener('resize', resizeHandler);
    resizeHandler = null;
  }
  if (socket) {
    socket.emit('leave-session');
    socket.disconnect();
    socket = null;
  }
  if (term) {
    term.dispose();
    term = null;
  }
  if (fitAddon) {
    fitAddon.dispose();
    fitAddon = null;
  }
  if (clipboardAddon) {
    clipboardAddon.dispose();
    clipboardAddon = null;
  }
  terminalSessionId = null;
  terminalActive.value = false;
};

onMounted(async () => {
  try {
    await fetchSettings();
    await checkStatus();
    if (!currentVersion.value) {
      checkForUpdates();
    }

    if (webuiEnabled.value) {
      statusInterval.value = setInterval(async () => {
        await checkStatus();
      }, 5000);
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  if (statusInterval.value) {
    clearInterval(statusInterval.value);
  }
  closeTerminal();
});
</script>
