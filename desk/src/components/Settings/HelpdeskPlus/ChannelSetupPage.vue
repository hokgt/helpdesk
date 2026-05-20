<template>
  <div class="flex max-h-[calc(100vh-2rem)] flex-col gap-6 overflow-y-auto p-6 pr-7">
    <div class="flex items-center justify-between gap-3">
      <div class="flex items-center gap-3">
        <component :is="PlugIcon" class="h-6 w-6 text-blue-600" />
        <h2 class="text-xl font-semibold text-gray-900">Channel Setup</h2>
      </div>
      <button
        v-if="!addingChannel"
        @click="startAddChannel()"
        :disabled="isSaving || availableChannels.length === 0"
        class="rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
      >
        Add Channel
      </button>
    </div>

    <p class="text-sm text-gray-600">
      Add one communication channel at a time. Pick a channel, test/verify the bound identity, then add it to the active channel records.
    </p>

    <div v-if="loading" class="flex items-center gap-2 py-8">
      <div class="h-5 w-5 animate-spin rounded-full border-2 border-gray-300 border-t-blue-600"></div>
      <span class="text-sm text-gray-500">Loading channel setup...</span>
    </div>

    <template v-else>
      <div class="rounded-lg border border-blue-100 bg-blue-50 p-4 text-sm text-blue-800">
        <strong>Workflow:</strong>
        Add Channel → choose WhatsApp, Telegram, or Mattermost → fill identity → Test Channel → Add Channel Record.
        Bot channels must already be bound in the selected gateway before they can be added.
      </div>

      <div v-if="configuredChannels.length" class="flex flex-col gap-3">
        <h3 class="text-sm font-semibold uppercase tracking-wide text-gray-500">Added Channels</h3>
        <div
          v-for="channel in configuredChannels"
          :key="channel.channel"
          class="rounded-lg border border-gray-200 bg-white p-4"
        >
          <div class="flex flex-wrap items-center justify-between gap-3">
            <div class="flex items-center gap-3">
              <component :is="channelIcon(channel.channel)" class="h-5 w-5" :class="channelIconClass(channel.channel)" />
              <div>
                <div class="font-medium text-gray-900">{{ channel.channel }}</div>
                <div class="text-sm text-green-700">Verified: {{ verifiedLabel(channel) }}</div>
                <div v-if="channel.channel === 'WhatsApp'" class="mt-1 text-xs" :class="whatsappLinkClass">
                  Link Status: {{ whatsappLinkLabel }}
                </div>
              </div>
            </div>
            <div class="flex flex-wrap gap-2">
              <button
                v-if="channel.channel === 'WhatsApp'"
                @click="toggleWhatsappPane()"
                :disabled="isSaving || isQrLoading"
                class="rounded-md border border-green-200 px-3 py-1.5 text-sm text-green-700 hover:bg-green-50 disabled:opacity-50"
              >
                {{ whatsappPaneOpen ? 'Close WhatsApp Link' : 'WhatsApp Link' }}
              </button>
              <button
                @click="editChannel(channel)"
                :disabled="isSaving || addingChannel"
                class="rounded-md border border-gray-300 px-3 py-1.5 text-sm text-gray-700 hover:bg-gray-50 disabled:opacity-50"
              >
                Edit
              </button>
              <button
                @click="removeChannel(channel)"
                :disabled="isSaving"
                class="rounded-md border border-red-200 px-3 py-1.5 text-sm text-red-600 hover:bg-red-50 disabled:opacity-50"
              >
                Remove
              </button>
            </div>
          </div>

          <div v-if="channel.channel === 'WhatsApp' && whatsappPaneOpen" class="mt-4 rounded-lg border border-green-100 bg-green-50 p-4">
            <div class="flex flex-wrap items-center justify-between gap-3">
              <div>
                <div class="text-sm font-semibold text-gray-900">Web WhatsApp Link</div>
                <div class="mt-1 text-sm" :class="whatsappLinkClass">{{ whatsappLinkLabel }}</div>
                <div v-if="whatsappLink.message" class="mt-1 text-xs text-gray-500">{{ whatsappLink.message }}</div>
              </div>
              <div class="flex flex-wrap gap-2">
                <button
                  @click="refreshWhatsappLinkStatus()"
                  :disabled="isQrLoading"
                  class="rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm text-gray-700 hover:bg-gray-50 disabled:opacity-50"
                >
                  Refresh Status
                </button>
                <button
                  @click="generateWhatsappQr(false)"
                  :disabled="isQrLoading"
                  class="rounded-md border border-green-300 bg-white px-3 py-1.5 text-sm text-green-700 hover:bg-green-100 disabled:opacity-50"
                >
                  {{ qrPanel.qr_image || qrPanel.qr_data ? 'Regenerate QR Code' : 'Generate WhatsApp QR Code' }}
                </button>
                <button
                  @click="generateWhatsappQr(true)"
                  :disabled="isQrLoading"
                  class="rounded-md border border-orange-300 bg-white px-3 py-1.5 text-sm text-orange-700 hover:bg-orange-50 disabled:opacity-50"
                >
                  Relink / Force New QR
                </button>
              </div>
            </div>

            <div v-if="!whatsappLink.is_linked && (qrPanel.qr_image || qrPanel.qr_data || qrPanel.message)" class="mt-4 flex flex-wrap items-start gap-4">
              <div v-if="qrPanel.qr_image" class="rounded-md border border-gray-200 bg-white p-3">
                <img :src="qrPanel.qr_image" alt="WhatsApp QR Code" class="h-48 w-48 object-contain" />
              </div>
              <div class="min-w-[220px] flex-1 text-sm text-gray-700">
                <p class="font-medium text-gray-900">{{ qrPanel.error ? 'QR code could not be generated.' : 'Scan with WhatsApp on your phone.' }}</p>
                <p v-if="!qrPanel.error" class="mt-1">Scan this QR code with WhatsApp. It will be hidden automatically after the link is connected.</p>
                <pre v-if="qrPanel.qr_data && !qrPanel.qr_image" class="mt-2 max-h-32 overflow-auto rounded bg-white p-2 text-xs text-gray-600">{{ qrPanel.qr_data }}</pre>
                <p v-if="qrPanel.message" class="mt-2 text-xs" :class="qrPanel.error ? 'text-red-600' : 'text-gray-500'">{{ qrPanel.message }}</p>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else-if="!addingChannel" class="rounded-lg border border-dashed border-gray-300 bg-white p-8 text-center">
        <component :is="PlugIcon" class="mx-auto mb-3 h-8 w-8 text-gray-400" />
        <h3 class="text-base font-semibold text-gray-900">No channels added yet</h3>
        <p class="mt-1 text-sm text-gray-500">Click Add Channel to configure the first communication channel.</p>
      </div>

      <div v-if="addingChannel" class="rounded-lg border border-gray-200 bg-white p-5">
        <div class="mb-4 flex flex-wrap items-start justify-between gap-3">
          <div>
            <h3 class="text-base font-semibold text-gray-900">{{ editMode ? 'Edit Channel' : 'Add Channel' }}</h3>
            <p class="text-xs text-gray-500">Test must pass before this channel can be saved as an active record.</p>
          </div>
          <button @click="cancelAddChannel" :disabled="isSaving" class="text-sm text-gray-500 hover:text-gray-700">Cancel</button>
        </div>

        <div class="grid grid-cols-1 gap-4 lg:grid-cols-2">
          <div>
            <label class="mb-1 block text-sm font-medium text-gray-600">Channel</label>
            <select
              v-model="draftChannel.channel"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              :disabled="editMode || isSaving"
              @change="onChannelSelected"
            >
              <option value="" disabled>Select channel...</option>
              <option v-for="channel in availableChannels" :key="channel.channel" :value="channel.channel">{{ channel.channel }}</option>
            </select>
            <p class="mt-1 text-xs text-gray-400">{{ channelDescription(draftChannel.channel) }}</p>
          </div>

          <div v-if="draftChannel.channel">
            <label class="mb-1 block text-sm font-medium text-gray-600">Gateway</label>
            <select
              v-model="draftChannel.gateway_type"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              :disabled="isSaving"
              @change="markUnverified(draftChannel)"
            >
              <option value="OpenClaw">OpenClaw</option>
              <option value="Hermes">Hermes</option>
            </select>
            <p class="mt-1 text-xs text-gray-400">Choose where this channel's bot/agent is already bound.</p>
          </div>

          <div v-if="draftChannel.channel">
            <label class="mb-1 block text-sm font-medium text-gray-600">Agent Name</label>
            <input
              v-model="draftChannel.agent_name"
              type="text"
              placeholder="helpdesk-bot"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              :disabled="isSaving"
              @input="markUnverified(draftChannel)"
            />
            <p class="mt-1 text-xs text-gray-400">Bound agent key in the selected gateway.</p>
          </div>

          <div v-if="draftChannel.channel">
            <label class="mb-1 block text-sm font-medium text-gray-600">Display Sender Name</label>
            <input
              v-model="draftChannel.display_sender_name"
              type="text"
              placeholder="Helpdesk Wijayacorp"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500"
              :disabled="isSaving"
            />
            <p class="mt-1 text-xs text-gray-400">Front-facing sender/user name shown to ticket posters.</p>
          </div>
        </div>

        <div class="mt-4 flex flex-wrap items-center gap-2">
          <button
            @click="testChannel"
            :disabled="isSaving || !draftChannel.channel"
            class="rounded-md border border-blue-200 px-4 py-2 text-sm font-medium text-blue-700 hover:bg-blue-50 disabled:opacity-50"
          >
            {{ isSaving ? 'Testing...' : 'Test Channel' }}
          </button>
          <button
            @click="saveDraftChannel"
            :disabled="isSaving || !canSaveDraft"
            class="rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
            {{ editMode ? 'Update Channel Record' : 'Add Channel Record' }}
          </button>

          <span v-if="isVerified(draftChannel)" class="text-sm text-green-700">Verified: {{ verifiedLabel(draftChannel) }}</span>
          <span v-else-if="draftChannel.verification_error" class="text-sm text-red-600">{{ draftChannel.verification_error }}</span>
          <span v-else class="text-sm text-gray-400">Not verified</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { computed, ref, onMounted, onBeforeUnmount } from "vue";
import LucidePlug from "~icons/lucide/plug";
import LucideMessageCircle from "~icons/lucide/message-circle";
import LucideSend from "~icons/lucide/send";
import LucideHash from "~icons/lucide/hash";

const PlugIcon = LucidePlug;
const WhatsAppIcon = LucideMessageCircle;
const TelegramIcon = LucideSend;
const MattermostIcon = LucideHash;

const channels = ref([]);
const loading = ref(true);
const isSaving = ref(false);
const addingChannel = ref(false);
const editMode = ref(false);
const draftChannel = ref(blankDraft());
const whatsappPaneOpen = ref(false);
const isQrLoading = ref(false);
const qrPollTimer = ref(null);
const whatsappLink = ref({ status: "unknown", is_linked: false, number: "", message: "" });
const qrPanel = ref({ qr_image: "", qr_data: "", message: "" });

function blankDraft() {
  return { channel: "", enabled: false, display_sender_name: "Helpdesk Wijayacorp", gateway_type: "Hermes" };
}

const configuredChannels = computed(() => channels.value.filter((channel) => isVerified(channel)));
const configuredChannelNames = computed(() => new Set(configuredChannels.value.map((channel) => channel.channel)));
const availableChannels = computed(() =>
  channels.value.filter((channel) => editMode.value || !configuredChannelNames.value.has(channel.channel))
);
const canSaveDraft = computed(() => isVerified(draftChannel.value));
const hasConfiguredWhatsApp = computed(() => configuredChannels.value.some((channel) => channel.channel === "WhatsApp"));
function formatWhatsAppNumber(number) {
  const raw = String(number || "").split("@")[0].split(":")[0];
  const digits = raw.replace(/\D/g, "");
  return digits ? `+${digits}` : "";
}

const whatsappLinkLabel = computed(() => {
  if (whatsappLink.value.is_linked) {
    const number = formatWhatsAppNumber(whatsappLink.value.number);
    return `Linked${number ? `: ${number}` : ""}`;
  }
  const status = whatsappLink.value.status || "unknown";
  return status === "unknown" ? "Unknown / not checked" : status.replace(/_/g, " ");
});
const whatsappLinkClass = computed(() => (whatsappLink.value.is_linked ? "text-green-700" : "text-orange-700"));

function channelIcon(channel) {
  return { WhatsApp: WhatsAppIcon, Telegram: TelegramIcon, Mattermost: MattermostIcon }[channel] || PlugIcon;
}

function channelIconClass(channel) {
  return { WhatsApp: "text-green-600", Telegram: "text-sky-600", Mattermost: "text-purple-600" }[channel] || "text-blue-600";
}

function channelDescription(channel) {
  return {
    WhatsApp: "Use an already-bound WhatsApp agent from OpenClaw or Hermes.",
    Telegram: "Use an already-bound Telegram bot agent from OpenClaw or Hermes.",
    Mattermost: "Use an already-bound Mattermost bot/user agent from OpenClaw or Hermes.",
  }[channel] || "Choose the channel to add.";
}

function currentIdentity(channel) {
  return channel.agent_name;
}

function isVerified(channel) {
  const identity = currentIdentity(channel);
  const sameGateway = channel._verified_gateway_type === channel.gateway_type;
  return Boolean(
    channel.enabled &&
      !channel.verification_error &&
      channel.is_linked &&
      channel.linked_identifier &&
      channel.linked_identifier === identity &&
      channel._verified_identity === identity &&
      sameGateway
  );
}

function markUnverified(channel) {
  channel.enabled = false;
  channel.is_linked = false;
  channel.linked_identifier = "";
  channel._verified_identity = "";
  channel._verified_gateway_type = "";
  channel.verification_error = "";
}

function verifiedLabel(channel) {
  const identity = channel.linked_identifier || channel.agent_name || "configured";
  return `${channel.gateway_type}: ${identity}`;
}

function hydrateChannel(channel) {
  return {
    ...channel,
    _verified_gateway_type: channel.is_linked ? channel.gateway_type : "",
    _verified_identity: channel.is_linked ? channel.linked_identifier : "",
  };
}

async function fetchSettings() {
  loading.value = true;
  try {
    const resp = await fetch("/api/method/helpdesk_plus.channel_setup_api.get_channel_settings");
    const data = await resp.json();
    channels.value = (data.message?.channels || []).map(hydrateChannel);
    if (hasConfiguredWhatsApp.value) {
      await refreshWhatsappLinkStatus(false);
    }
  } catch (e) {
    console.error("Failed to fetch channel settings:", e);
    window.frappe?.show_alert?.({ message: "Failed to load channel setup", indicator: "red" });
  } finally {
    loading.value = false;
  }
}

function getErrorMessage(data, fallback) {
  const exc = data?._server_messages;
  if (exc) {
    try {
      const messages = JSON.parse(exc).map((msg) => JSON.parse(msg).message || msg);
      return messages.join("\n");
    } catch (e) {
      return fallback;
    }
  }
  return data?.message?.error || fallback;
}

function startAddChannel() {
  addingChannel.value = true;
  editMode.value = false;
  draftChannel.value = blankDraft();
}

function editChannel(channel) {
  addingChannel.value = true;
  editMode.value = true;
  draftChannel.value = hydrateChannel({ ...channel });
}

function cancelAddChannel() {
  addingChannel.value = false;
  editMode.value = false;
  draftChannel.value = blankDraft();
}

function onChannelSelected() {
  const source = channels.value.find((channel) => channel.channel === draftChannel.value.channel);
  draftChannel.value = hydrateChannel({ ...source, enabled: false, is_linked: false, linked_identifier: "", verification_error: "" });
}

async function testChannel() {
  isSaving.value = true;
  try {
    const payload = { ...draftChannel.value, enabled: true };
    const resp = await fetch("/api/method/helpdesk_plus.channel_setup_api.test_channel_config", {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-Frappe-CSRF-Token": window.csrf_token || "" },
      body: JSON.stringify({ channel: payload }),
    });
    const data = await resp.json();
    if (!resp.ok || !data.message?.ok) {
      throw new Error(getErrorMessage(data, "Channel test failed"));
    }
    draftChannel.value.enabled = true;
    draftChannel.value.is_linked = true;
    draftChannel.value.linked_identifier = data.message.linked_identifier || currentIdentity(draftChannel.value);
    draftChannel.value._verified_identity = draftChannel.value.linked_identifier;
    draftChannel.value._verified_gateway_type = draftChannel.value.gateway_type;
    draftChannel.value.verification_error = "";
    window.frappe?.show_alert?.({ message: data.message.message || "Channel verified", indicator: "green" });
  } catch (e) {
    markUnverified(draftChannel.value);
    draftChannel.value.verification_error = e.message || "Channel test failed";
    window.frappe?.show_alert?.({ message: draftChannel.value.verification_error, indicator: "red" });
  } finally {
    isSaving.value = false;
  }
}

async function saveChannel(channelToSave, successMessage = "Channel setup saved") {
  const resp = await fetch("/api/method/helpdesk_plus.channel_setup_api.save_channel_settings", {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-Frappe-CSRF-Token": window.csrf_token || "" },
    body: JSON.stringify({ channels: [channelToSave] }),
  });
  const data = await resp.json();
  if (!resp.ok || !data.message?.ok) {
    throw new Error(getErrorMessage(data, "Save failed"));
  }
  window.frappe?.show_alert?.({ message: successMessage, indicator: "green" });
}

async function saveDraftChannel() {
  if (!canSaveDraft.value) return;
  isSaving.value = true;
  try {
    await saveChannel({ ...draftChannel.value, enabled: true }, editMode.value ? "Channel record updated" : "Channel record added");
    cancelAddChannel();
    await fetchSettings();
  } catch (e) {
    draftChannel.value.verification_error = e.message || "Failed to save channel setup";
    window.frappe?.show_alert?.({ message: draftChannel.value.verification_error, indicator: "red" });
    await fetchSettings();
  } finally {
    isSaving.value = false;
  }
}

async function removeChannel(channel) {
  isSaving.value = true;
  try {
    await saveChannel({ ...channel, enabled: false, is_linked: false, linked_identifier: "" }, "Channel removed");
    if (draftChannel.value.channel === channel.channel) {
      cancelAddChannel();
    }
    if (channel.channel === "WhatsApp") {
      whatsappPaneOpen.value = false;
      stopWhatsappPolling();
      qrPanel.value = { qr_image: "", qr_data: "", message: "", error: false };
    }
    await fetchSettings();
  } catch (e) {
    window.frappe?.show_alert?.({ message: e.message || "Failed to remove channel", indicator: "red" });
  } finally {
    isSaving.value = false;
  }
}

function toggleWhatsappPane() {
  whatsappPaneOpen.value = !whatsappPaneOpen.value;
  if (whatsappPaneOpen.value) {
    refreshWhatsappLinkStatus();
  } else {
    stopWhatsappPolling();
  }
}

async function refreshWhatsappLinkStatus(showError = true) {
  if (!hasConfiguredWhatsApp.value) return;
  try {
    const resp = await fetch("/api/method/helpdesk_plus.channel_setup_api.get_whatsapp_link_status");
    const data = await resp.json();
    if (!resp.ok || !data.message?.ok) {
      throw new Error(getErrorMessage(data, "Failed to check WhatsApp link status"));
    }
    whatsappLink.value = { ...data.message, number: formatWhatsAppNumber(data.message?.number) };
    if (whatsappLink.value.is_linked) {
      qrPanel.value = { qr_image: "", qr_data: "", message: "", error: false };
      stopWhatsappPolling();
    }
    if (data.message.is_linked) {
      stopWhatsappPolling();
    }
  } catch (e) {
    if (showError) {
      window.frappe?.show_alert?.({ message: e.message || "Failed to check WhatsApp link status", indicator: "red" });
    }
  }
}

async function generateWhatsappQr(force = false) {
  isQrLoading.value = true;
  try {
    const resp = await fetch("/api/method/helpdesk_plus.channel_setup_api.generate_whatsapp_qr", {
      method: "POST",
      headers: { "Content-Type": "application/json", "X-Frappe-CSRF-Token": window.csrf_token || "" },
      body: JSON.stringify({ force }),
    });
    const data = await resp.json();
    if (!resp.ok || !data.message?.ok) {
      throw new Error(getErrorMessage(data, "Failed to generate WhatsApp QR code"));
    }
    if (data.message.is_linked) {
      whatsappLink.value = { ...data.message, number: formatWhatsAppNumber(data.message.number) };
      qrPanel.value = { qr_image: "", qr_data: "", message: "", error: false };
      stopWhatsappPolling();
    } else {
      qrPanel.value = {
        qr_image: data.message.qr_image || "",
        qr_data: data.message.qr_data || "",
        message: data.message.message || "Scan the QR code with WhatsApp to link this channel.",
        error: false,
      };
      whatsappLink.value = { ...whatsappLink.value, status: data.message.status || "pending", is_linked: false };
      startWhatsappPolling();
    }
    window.frappe?.show_alert?.({ message: "WhatsApp QR code generated", indicator: "green" });
  } catch (e) {
    const message = e.message || "Failed to generate WhatsApp QR code";
    qrPanel.value = { qr_image: "", qr_data: "", message, error: true };
    window.frappe?.show_alert?.({ message, indicator: "red" });
  } finally {
    isQrLoading.value = false;
  }
}

function startWhatsappPolling() {
  stopWhatsappPolling();
  qrPollTimer.value = window.setInterval(() => refreshWhatsappLinkStatus(false), 5000);
}

function stopWhatsappPolling() {
  if (qrPollTimer.value) {
    window.clearInterval(qrPollTimer.value);
    qrPollTimer.value = null;
  }
}

onMounted(fetchSettings);
onBeforeUnmount(stopWhatsappPolling);
</script>
