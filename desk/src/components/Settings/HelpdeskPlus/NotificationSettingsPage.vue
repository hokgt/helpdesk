<template>
  <div class="flex max-h-[calc(100vh-2rem)] flex-col gap-6 overflow-y-auto p-6 pr-7">
    <div class="flex items-center gap-3">
      <component :is="BellIcon" class="h-6 w-6 text-blue-600" />
      <h2 class="text-xl font-semibold text-gray-900">Poster Notifications</h2>
    </div>

    <p class="text-sm text-gray-600">
      Configure automatic notifications sent to ticket posters when their ticket
      status changes. Messages can be delivered through WhatsApp, Mattermost,
      or Telegram.
    </p>

    <div v-if="loading" class="flex items-center gap-2 py-8">
      <div class="h-5 w-5 animate-spin rounded-full border-2 border-gray-300 border-t-blue-600"></div>
      <span class="text-sm text-gray-500">Loading settings...</span>
    </div>

    <template v-else>
      <div class="rounded-lg border border-gray-200 bg-white p-5">
        <h3 class="mb-4 text-sm font-medium text-gray-700">Global Settings</h3>

        <div class="mb-4">
          <label class="inline-flex items-center gap-2">
            <input type="checkbox" v-model="settings.enabled" class="rounded" @change="scheduleGlobalSave" />
            <span class="text-sm text-gray-700" title="Master switch. When off, no poster notification is sent from these triggers.">Enable Notifications</span>
          </label>
          <p class="mt-1 text-xs text-gray-500">
            Master switch. When off, no poster notification is sent from these triggers.
          </p>
        </div>

        <div class="mb-4">
          <div class="mb-2 flex items-center justify-between">
            <label class="block text-sm font-medium text-gray-600">Notification Channels</label>
            <button
              type="button"
              @click="addChannel"
              :disabled="!availableChannels.length || settings.notification_channels.length >= 3"
              class="rounded-md border border-gray-300 px-2 py-1 text-xs text-gray-600 hover:bg-gray-50 disabled:cursor-not-allowed disabled:opacity-50"
            >
              Add Channel
            </button>
          </div>
          <div
            v-if="!availableChannels.length"
            class="rounded-md border border-amber-200 bg-amber-50 px-3 py-2 text-sm text-amber-800"
          >
            No notification channel has been setup yet. Please setup and enable at least one channel in
            <strong>Channel Setup</strong> before adding Notification Triggers channels.
          </div>

          <div v-else class="flex flex-col gap-2">
            <div
              v-for="(channel, index) in settings.notification_channels"
              :key="index"
              class="grid grid-cols-1 gap-2 sm:grid-cols-[1fr_1fr_auto]"
            >
              <select
                v-model="channel.channel"
                class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
                @change="onChannelChanged"
              >
                <option
                  v-for="option in availableChannelOptions(channel.channel)"
                  :key="option"
                  :value="option"
                >
                  {{ option }}
                </option>
              </select>

              <div
                v-if="index === 0"
                class="flex items-center rounded-md border border-gray-200 bg-gray-50 px-3 py-2 text-sm text-gray-700"
              >
                Main Channel
              </div>
              <select
                v-else
                v-model="channel.delivery_mode"
                class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
                @change="scheduleGlobalSave"
              >
                <option value="Additional">Additional</option>
                <option value="Fallback">Fallback</option>
              </select>

              <button
                type="button"
                @click="removeChannel(index)"
                :disabled="index === 0 || settings.notification_channels.length <= 1"
                class="rounded-md border border-red-200 px-3 py-2 text-xs text-red-600 hover:bg-red-50 disabled:cursor-not-allowed disabled:opacity-40"
              >
                Remove
              </button>
            </div>
          </div>
          <p class="mt-1 text-xs text-gray-500">
            Only channels enabled in Channel Setup can be selected here. The first row is always the Main channel. Additional sends another copy; Fallback is used only if the main/fallback route before it fails.
          </p>
        </div>

        <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
          <div>
            <label class="mb-1 block text-sm font-medium text-gray-600">Default Language</label>
            <select
              v-model="settings.default_language"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              @change="onDefaultLanguageChanged"
            >
              <option value="en">English</option>
              <option value="id">Indonesian</option>
            </select>
            <p class="mt-1 text-xs text-gray-500">Changing this loads the templates saved for that language.</p>
          </div>

          <div>
            <label class="mb-1 block text-sm font-medium text-gray-600" title="Optional delay before sending a notification. Use 0 to send immediately.">Send Delay (seconds) <span class="cursor-help text-gray-400">ⓘ</span></label>
            <input
              v-model.number="settings.default_send_delay_seconds"
              type="number"
              min="0"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              @change="scheduleGlobalSave"
            />
          </div>

          <div>
            <label class="mb-1 block text-sm font-medium text-gray-600" title="Suppress duplicate notifications for the same ticket and trigger within this many minutes.">Dedupe Window (minutes) <span class="cursor-help text-gray-400">ⓘ</span></label>
            <input
              v-model.number="settings.default_dedupe_window_minutes"
              type="number"
              min="0"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              @change="scheduleGlobalSave"
            />
          </div>

          <label class="flex items-center gap-2">
            <input type="checkbox" v-model="settings.reminder_enabled" class="rounded" @change="scheduleGlobalSave" />
            <span class="text-sm text-gray-700" title="When enabled, the system can send follow-up reminder notifications after the configured reminder time if the ticket still needs action.">Enable Reminders <span class="cursor-help text-gray-400">ⓘ</span></span>
          </label>

          <div v-if="settings.reminder_enabled">
            <label class="mb-1 block text-sm font-medium text-gray-600" title="How many hours to wait before sending a reminder when reminders are enabled.">Reminder After (hours) <span class="cursor-help text-gray-400">ⓘ</span></label>
            <input
              v-model.number="settings.default_reminder_after_hours"
              type="number"
              min="1"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              @change="scheduleGlobalSave"
            />
          </div>

          <label class="flex items-center gap-2">
            <input type="checkbox" v-model="settings.audit_enabled" class="rounded" @change="scheduleGlobalSave" />
            <span class="text-sm text-gray-700" title="Record notification attempts and results in the notification log for troubleshooting and audit history.">Enable Audit Log <span class="cursor-help text-gray-400">ⓘ</span></span>
          </label>
        </div>

        <p class="mt-4 text-xs text-gray-500">
          {{ isSaving ? 'Saving changes...' : 'Changes are applied automatically.' }}
        </p>
      </div>

      <div class="rounded-lg border border-gray-200 bg-white p-5">
        <div class="mb-4 flex items-center justify-between">
          <div>
            <h3 class="text-sm font-medium text-gray-700">Trigger Templates</h3>
            <p class="mt-1 text-xs text-gray-500">
              Templates are universal for all channels. Each language keeps its own saved message text.
            </p>
          </div>
          <div class="flex items-center gap-2">
            <button
              @click="showVariableLegend = true"
              class="rounded-md border border-blue-200 px-3 py-1.5 text-xs text-blue-700 hover:bg-blue-50"
            >
              Variable Legend
            </button>
            <button
              @click="resetDefaults"
              class="rounded-md border border-gray-300 px-3 py-1.5 text-xs text-gray-600 hover:bg-gray-50"
            >
              Reset to Defaults
            </button>
          </div>
        </div>

        <div class="flex flex-col gap-4">
          <div
            v-for="trigger in settings.trigger_templates"
            :key="trigger.trigger_key"
            class="rounded-lg border p-4"
            :class="trigger.enabled ? 'border-blue-200 bg-blue-50/30' : 'border-gray-200 bg-gray-50'"
          >
            <div class="mb-3 flex items-center justify-between">
              <div class="flex items-center gap-3">
                <label class="flex items-center gap-2">
                  <input
                    type="checkbox"
                    v-model="trigger.enabled"
                    class="rounded"
                    @change="saveTrigger(trigger)"
                  />
                  <span class="text-sm font-medium text-gray-800">{{ trigger.trigger_label }}</span>
                </label>
              </div>
              <button
                @click="toggleExpand(trigger.trigger_key)"
                class="text-xs text-blue-600 hover:text-blue-800"
              >
                {{ expandedTriggers[trigger.trigger_key] ? 'Collapse' : 'Edit' }}
              </button>
            </div>

            <div v-if="expandedTriggers[trigger.trigger_key]" class="mt-3 flex flex-col gap-3">
              <div class="w-full sm:w-1/2">
                <label class="mb-1 block text-xs font-medium text-gray-600">Format</label>
                <select
                  v-model="trigger.message_format"
                  class="w-full rounded-md border border-gray-300 px-2 py-1.5 text-sm"
                >
                  <option value="text">Plain Text</option>
                  <option value="markdown">Markdown</option>
                </select>
              </div>

              <div>
                <label class="mb-1 block text-xs font-medium text-gray-600">
                  Message Template — {{ languageLabel(settings.default_language) }}
                  <button
                    type="button"
                    @click="showVariableLegend = true"
                    class="ml-2 text-xs font-normal text-blue-600 hover:text-blue-800"
                  >
                    View variables
                  </button>
                </label>
                <textarea
                  v-model="trigger.message_template"
                  rows="3"
                  class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm font-mono"
                ></textarea>
              </div>

              <div class="flex gap-2">
                <button
                  @click="saveTrigger(trigger)"
                  :disabled="isSaving"
                  class="rounded-md bg-blue-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-blue-700 disabled:opacity-50"
                >
                  Save Trigger
                </button>
                <button
                  @click="sendTest(trigger)"
                  :disabled="isTesting"
                  class="rounded-md border border-green-300 px-3 py-1.5 text-sm font-medium text-green-700 hover:bg-green-50 disabled:opacity-50"
                >
                  {{ isTesting ? 'Sending...' : 'Send Test' }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div
        v-if="showTestDialog"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 p-4"
        @click.self="closeTestDialog"
      >
        <div class="w-full max-w-lg rounded-lg bg-white p-5 shadow-xl">
          <div class="mb-4 flex items-start justify-between gap-4">
            <div>
              <h3 class="text-base font-semibold text-gray-900">Send Test Notification</h3>
              <p class="mt-1 text-xs text-gray-500">
                Select one or more configured platforms to send this test message simultaneously.
              </p>
            </div>
            <button class="text-sm text-gray-500 hover:text-gray-800" @click="closeTestDialog">Close</button>
          </div>

          <div class="mb-4 flex flex-col gap-2">
            <label
              v-for="channel in configuredTestChannels()"
              :key="channel"
              class="flex items-center gap-2 rounded-md border border-gray-200 px-3 py-2 text-sm"
            >
              <input
                type="checkbox"
                :value="channel"
                v-model="selectedTestChannels"
                class="rounded"
              />
              <span>{{ channel }}</span>
            </label>
          </div>

          <div v-if="selectedTestChannels.includes('WhatsApp')" class="mb-4">
            <label class="mb-1 block text-xs font-medium text-gray-600">WhatsApp test phone number</label>
            <input
              v-model="testPhone"
              type="text"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              placeholder="With country code, e.g. 628123456789"
            />
            <p class="mt-1 text-xs text-gray-500">Required only when WhatsApp is selected.</p>
          </div>

          <div v-if="selectedTestChannels.includes('Mattermost')" class="mb-4">
            <label class="mb-1 block text-xs font-medium text-gray-600">Mattermost target user</label>
            <input
              v-model="testMattermostTarget"
              type="text"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              placeholder="Username, e.g. hok"
            />
            <p class="mt-1 text-xs text-gray-500">Required when Mattermost is selected. Enter username without @.</p>
          </div>

          <div v-if="selectedTestChannels.includes('Telegram')" class="mb-4">
            <label class="mb-1 block text-xs font-medium text-gray-600">Telegram target chat/user</label>
            <input
              v-model="testTelegramTarget"
              type="text"
              class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
              placeholder="Chat ID or @channelusername"
            />
            <p class="mt-1 text-xs text-gray-500">Required when Telegram is selected. Use a chat ID or public @channelusername reachable by the bot.</p>
          </div>

          <div v-if="testResult" class="mb-4 rounded-md border px-3 py-2 text-sm" :class="testResult.ok ? 'border-green-200 bg-green-50 text-green-700' : 'border-red-200 bg-red-50 text-red-700'">
            <div class="font-medium">{{ testResult.ok ? 'Test sent' : 'Test failed' }}</div>
            <ul class="mt-1 list-disc pl-5 text-xs">
              <li v-for="result in testResult.results || []" :key="result.channel">
                {{ result.channel }}: {{ result.status || (result.ok ? 'Sent' : 'Failed') }}{{ result.error ? ` — ${result.error}` : '' }}
              </li>
            </ul>
          </div>

          <div class="flex justify-end gap-2">
            <button
              type="button"
              class="rounded-md border border-gray-300 px-3 py-1.5 text-sm text-gray-700 hover:bg-gray-50"
              @click="closeTestDialog"
            >
              Cancel
            </button>
            <button
              type="button"
              class="rounded-md bg-green-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-green-700 disabled:opacity-50"
              :disabled="isTesting || !selectedTestChannels.length || (selectedTestChannels.includes('WhatsApp') && !testPhone.trim()) || (selectedTestChannels.includes('Mattermost') && !testMattermostTarget.trim()) || (selectedTestChannels.includes('Telegram') && !testTelegramTarget.trim())"
              @click="sendSelectedTest"
            >
              {{ isTesting ? 'Sending...' : 'Send Test' }}
            </button>
          </div>
        </div>
      </div>

      <div
        v-if="showVariableLegend"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 p-4"
        @click.self="showVariableLegend = false"
      >
        <div class="max-h-[80vh] w-full max-w-3xl overflow-y-auto rounded-lg bg-white p-5 shadow-xl">
          <div class="mb-4 flex items-start justify-between gap-4">
            <div>
              <h3 class="text-base font-semibold text-gray-900">Trigger Template Variable Legend</h3>
              <p class="mt-1 text-xs text-gray-500">
                Use variables inside templates with Jinja syntax, for example <code v-pre class="rounded bg-gray-100 px-1">{{ ticket_id }}</code>.
              </p>
            </div>
            <button class="text-sm text-gray-500 hover:text-gray-800" @click="showVariableLegend = false">Close</button>
          </div>
          <div class="overflow-x-auto">
            <table class="w-full text-left text-sm">
              <thead>
                <tr class="border-b text-xs uppercase text-gray-500">
                  <th class="pb-2 pr-4">Variable</th>
                  <th class="pb-2">Description</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in variableLegend" :key="item.variable" class="border-b border-gray-100">
                  <td class="py-2 pr-4 align-top font-mono text-xs text-blue-700">{{ item.variable }}</td>
                  <td class="py-2 text-gray-700">{{ item.description }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <div class="rounded-lg border border-gray-200 bg-white p-5">
        <div class="mb-4 flex items-center justify-between">
          <h3 class="text-sm font-medium text-gray-700">Recent Notification Log</h3>
          <button
            @click="fetchLogs"
            class="rounded-md border border-gray-300 px-3 py-1.5 text-xs text-gray-600 hover:bg-gray-50"
          >
            Refresh
          </button>
        </div>

        <div v-if="logs.length === 0" class="py-4 text-center text-sm text-gray-400">
          No notifications sent yet.
        </div>

        <div v-else class="overflow-x-auto">
          <table class="w-full text-left text-sm">
            <thead>
              <tr class="border-b text-xs text-gray-500">
                <th class="pb-2 pr-4">Ticket</th>
                <th class="pb-2 pr-4">Trigger</th>
                <th class="pb-2 pr-4">Channel</th>
                <th class="pb-2 pr-4">Recipient</th>
                <th class="pb-2 pr-4">Status</th>
                <th class="pb-2 pr-4">Sent At</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="log in logs" :key="log.name" class="border-b border-gray-100">
                <td class="py-2 pr-4 font-mono text-xs">{{ log.ticket }}</td>
                <td class="py-2 pr-4">{{ log.trigger_label || log.trigger_key }}</td>
                <td class="py-2 pr-4">{{ log.channel }}</td>
                <td class="py-2 pr-4">{{ log.recipient_name || log.recipient_phone }}</td>
                <td class="py-2 pr-4">
                  <span
                    class="rounded-full px-2 py-0.5 text-xs font-medium"
                    :class="{
                      'bg-green-100 text-green-700': log.status === 'Sent',
                      'bg-red-100 text-red-700': log.status === 'Failed',
                      'bg-yellow-100 text-yellow-700': log.status === 'Queued',
                      'bg-gray-100 text-gray-600': log.status === 'Skipped',
                    }"
                  >
                    {{ log.status }}
                  </span>
                </td>
                <td class="py-2 pr-4 text-xs text-gray-500">{{ formatDate(log.sent_at || log.creation) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, watch } from "vue";
import LucideBell from "~icons/lucide/bell";

const BellIcon = LucideBell;

const loading = ref(true);
const isSaving = ref(false);
const isTesting = ref(false);
const isLoadingLanguage = ref(false);
const showVariableLegend = ref(false);
const showTestDialog = ref(false);
const selectedTestChannels = ref([]);
const testPhone = ref("");
const testMattermostTarget = ref("");
const testTelegramTarget = ref("");
const testTrigger = ref(null);
const testResult = ref(null);
let globalSaveTimer = null;

const settings = reactive({
  enabled: false,
  default_channel: "WhatsApp",
  default_language: "en",
  notification_channels: [],
  available_channels: [],
  reminder_enabled: false,
  default_reminder_after_hours: 24,
  default_send_delay_seconds: 0,
  default_dedupe_window_minutes: 5,
  audit_enabled: true,
  trigger_templates: [],
});

const expandedTriggers = reactive({});
const logs = ref([]);
const availableChannels = ref([]);

function uniqueChannels(rows) {
  return [...new Set((rows || []).map((row) => row.channel).filter(Boolean))];
}

function configuredTestChannels() {
  return uniqueChannels(availableChannels.value.map((channel) => ({ channel })));
}

const variableLegend = [
  { variable: "{{ ticket_id }}", description: "Ticket ID/name shown to the poster, for example HD-0001 or TEST-001." },
  { variable: "{{ ticket_subject }}", description: "Ticket subject/title." },
  { variable: "{{ poster_name }}", description: "Name of the ticket poster/customer when available." },
  { variable: "{{ poster_email }}", description: "Email address of the ticket poster/customer when available." },
  { variable: "{{ agent_name }}", description: "Assigned/helpdesk agent name used in the notification context." },
  { variable: "{{ status }}", description: "Current ticket status or status label related to the trigger." },
  { variable: "{{ site_url }}", description: "Base URL of the Helpdesk site." },
  { variable: "{{ ticket.name }}", description: "Raw ticket document name. Usually the same value as ticket_id." },
  { variable: "{{ ticket.subject }}", description: "Raw ticket subject from the ticket document." },
  { variable: "{{ ticket.status }}", description: "Raw current ticket status from the ticket document." },
  { variable: "{{ poster.name }}", description: "Poster/customer display name." },
  { variable: "{{ poster.email }}", description: "Poster/customer email address." },
  { variable: "{{ uat.confirm_url }}", description: "UAT confirmation URL for resolved/user-testing notifications, when available." },
];

function languageLabel(language) {
  return language === "id" ? "Indonesian" : "English";
}

function normalizeChannels() {
  const allowed = availableChannels.value || [];
  if (!allowed.length) {
    settings.notification_channels = [];
    return;
  }

  const normalized = (settings.notification_channels || [])
    .filter((row) => allowed.includes(row.channel))
    .slice(0, allowed.length)
    .map((row, index) => ({
      channel: row.channel || allowed[index] || allowed[0],
      delivery_mode: index === 0 ? "Main" : (row.delivery_mode === "Fallback" ? "Fallback" : "Additional"),
    }));

  if (!normalized.length) {
    normalized.push({ channel: allowed[0], delivery_mode: "Main" });
  }
  settings.notification_channels = normalized;
}

function availableChannelOptions(current) {
  const selected = settings.notification_channels
    .map((row) => row.channel)
    .filter((channel) => channel && channel !== current);
  return availableChannels.value.filter((channel) => !selected.includes(channel));
}

function addChannel() {
  if (!availableChannels.value.length || settings.notification_channels.length >= availableChannels.value.length) return;
  const next = availableChannels.value.find(
    (channel) => !settings.notification_channels.some((row) => row.channel === channel)
  );
  if (next) {
    settings.notification_channels.push({ channel: next, delivery_mode: "Additional" });
    normalizeChannels();
    scheduleGlobalSave();
  }
}

function removeChannel(index) {
  if (index === 0 || settings.notification_channels.length <= 1) return;
  settings.notification_channels.splice(index, 1);
  normalizeChannels();
  scheduleGlobalSave();
}

function onChannelChanged() {
  normalizeChannels();
  scheduleGlobalSave();
}

function csrfHeaders() {
  return {
    "Content-Type": "application/json",
    "X-Frappe-CSRF-Token": window.csrf_token || "",
  };
}

async function fetchSettings(language = null) {
  loading.value = true;
  try {
    const params = language ? `?${new URLSearchParams({ language })}` : "";
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.get_notification_settings" + params
    );
    const data = await resp.json();
    if (data.message) {
      Object.assign(settings, data.message);
      availableChannels.value = data.message.available_channels || [];
      normalizeChannels();
    }
  } catch (e) {
    console.error("Failed to load notification settings:", e);
  } finally {
    loading.value = false;
  }
}

function scheduleGlobalSave() {
  if (loading.value || isLoadingLanguage.value) return;
  if (globalSaveTimer) clearTimeout(globalSaveTimer);
  globalSaveTimer = setTimeout(() => saveGlobalSettings({ silent: true }), 350);
}

async function onDefaultLanguageChanged() {
  if (globalSaveTimer) clearTimeout(globalSaveTimer);
  await saveGlobalSettings({ silent: true });
  isLoadingLanguage.value = true;
  await fetchSettings(settings.default_language);
  isLoadingLanguage.value = false;
}

async function saveGlobalSettings({ silent = false } = {}) {
  normalizeChannels();
  isSaving.value = true;
  try {
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.save_notification_settings",
      {
        method: "POST",
        headers: csrfHeaders(),
        body: JSON.stringify({
          enabled: settings.enabled ? 1 : 0,
          default_channel: settings.notification_channels[0]?.channel || settings.default_channel,
          default_language: settings.default_language,
          notification_channels: settings.notification_channels,
          reminder_enabled: settings.reminder_enabled ? 1 : 0,
          default_reminder_after_hours: settings.default_reminder_after_hours,
          default_send_delay_seconds: settings.default_send_delay_seconds,
          default_dedupe_window_minutes: settings.default_dedupe_window_minutes,
          audit_enabled: settings.audit_enabled ? 1 : 0,
        }),
      }
    );
    const data = await resp.json();
    if (data.message && data.message.ok && !silent) {
      window.frappe?.show_alert?.({ message: "Settings saved", indicator: "green" });
    }
  } catch (e) {
    console.error("Failed to save settings:", e);
    window.frappe?.show_alert?.({ message: "Failed to save settings", indicator: "red" });
  } finally {
    isSaving.value = false;
  }
}

async function saveTrigger(trigger) {
  isSaving.value = true;
  try {
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.save_trigger_template",
      {
        method: "POST",
        headers: csrfHeaders(),
        body: JSON.stringify({
          trigger_key: trigger.trigger_key,
          enabled: trigger.enabled ? 1 : 0,
          message_format: trigger.message_format,
          language: settings.default_language,
          message_template: trigger.message_template,
        }),
      }
    );
    const data = await resp.json();
    if (data.message && data.message.ok) {
      window.frappe?.show_alert?.({ message: "Trigger saved", indicator: "green" });
    }
  } catch (e) {
    console.error("Failed to save trigger:", e);
    window.frappe?.show_alert?.({ message: "Failed to save trigger", indicator: "red" });
  } finally {
    isSaving.value = false;
  }
}

async function resetDefaults() {
  if (!confirm("Reset all trigger templates to defaults? Current customizations will be lost.")) return;
  try {
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.reset_trigger_defaults",
      { method: "POST", headers: csrfHeaders() }
    );
    const data = await resp.json();
    if (data.message && data.message.ok) {
      await fetchSettings(settings.default_language);
      window.frappe?.show_alert?.({ message: "Triggers reset to defaults", indicator: "green" });
    }
  } catch (e) {
    console.error("Failed to reset defaults:", e);
  }
}

function sendTest(trigger) {
  testTrigger.value = trigger;
  selectedTestChannels.value = configuredTestChannels();
  testResult.value = null;
  showTestDialog.value = true;
}

function closeTestDialog() {
  if (isTesting.value) return;
  showTestDialog.value = false;
  testTrigger.value = null;
  testResult.value = null;
}

async function sendSelectedTest() {
  if (!testTrigger.value || !selectedTestChannels.value.length) return;
  if (selectedTestChannels.value.includes("WhatsApp") && !testPhone.value.trim()) return;
  if (selectedTestChannels.value.includes("Mattermost") && !testMattermostTarget.value.trim()) return;
  if (selectedTestChannels.value.includes("Telegram") && !testTelegramTarget.value.trim()) return;
  isTesting.value = true;
  testResult.value = null;
  try {
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.send_test_notification",
      {
        method: "POST",
        headers: csrfHeaders(),
        body: JSON.stringify({
          trigger_key: testTrigger.value.trigger_key,
          phone: testPhone.value,
          mattermost_target: testMattermostTarget.value,
          telegram_target: testTelegramTarget.value,
          channels: selectedTestChannels.value,
        }),
      }
    );
    const data = await resp.json();
    testResult.value = data.message || { ok: false, error: "Failed to send test" };
    if (testResult.value.ok) {
      window.frappe?.show_alert?.({ message: "Test notification sent!", indicator: "green" });
      await fetchLogs();
    } else {
      window.frappe?.show_alert?.({
        message: testResult.value.error || "Failed to send test",
        indicator: "red",
      });
    }
  } catch (e) {
    console.error("Test send failed:", e);
    testResult.value = { ok: false, error: "Test send failed", results: [] };
    window.frappe?.show_alert?.({ message: "Test send failed", indicator: "red" });
  } finally {
    isTesting.value = false;
  }
}

async function fetchLogs() {
  try {
    const resp = await fetch(
      "/api/method/helpdesk_plus.notification_api.get_notification_logs?" +
        new URLSearchParams({ limit: "20" })
    );
    const data = await resp.json();
    if (data.message && data.message.logs) {
      logs.value = data.message.logs;
    }
  } catch (e) {
    console.error("Failed to fetch logs:", e);
  }
}

function toggleExpand(key) {
  expandedTriggers[key] = !expandedTriggers[key];
}

function formatDate(dt) {
  if (!dt) return "-";
  return new Date(dt).toLocaleString();
}

watch(
  () => settings.default_language,
  async (language, oldLanguage) => {
    if (!oldLanguage || language === oldLanguage || isLoadingLanguage.value) return;
    await onDefaultLanguageChanged();
  }
);

onMounted(() => {
  fetchSettings();
  fetchLogs();
});
</script>
