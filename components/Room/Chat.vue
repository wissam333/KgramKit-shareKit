<template>
  <div class="chat-wrap">
    <div
      ref="messagesEl"
      class="messages-list"
      :dir="$i18n.locale === 'ar' ? 'ltr' : 'rtl'"
    >
      <div v-if="!messages.length" class="chat-empty">
        <div class="empty-icon-wrap">
          <Icon name="mdi:chat-outline" size="28" />
        </div>
        <p class="empty-title">{{ $t("noChatYet") }}</p>
        <p class="empty-sub">Start a conversation below</p>
      </div>

      <template v-for="msg in messages" :key="msg.id">
        <div class="msg-row" :class="{ mine: msg.fromMe }">
          <div class="msg-avatar" :class="{ me: msg.fromMe }">
            {{ (msg.name || "?").charAt(0).toUpperCase() }}
          </div>

          <div class="msg-content">
            <div class="msg-meta" :class="{ 'meta-mine': msg.fromMe }">
              <span class="msg-author">{{
                msg.fromMe ? $t("you") : msg.name
              }}</span>
              <span class="msg-time">{{ formatTime(msg.time) }}</span>
            </div>

            <div
              v-if="msg.type === 'chat'"
              class="msg-bubble"
              :class="{ 'bubble-mine': msg.fromMe }"
            >
              {{ msg.text }}
              <span class="bubble-tick" v-if="msg.fromMe">
                <Icon name="mdi:check-all" size="12" />
              </span>
            </div>

            <div
              v-else-if="msg.type === 'file'"
              class="msg-file"
              :class="{ 'file-mine': msg.fromMe }"
            >
              <template v-if="msg.isImage && msg.url">
                <div class="img-wrapper" @click="openImage(msg.url)">
                  <img :src="msg.url" :alt="msg.fileName" class="msg-image" />
                  <div class="img-overlay">
                    <Icon name="mdi:magnify-plus-outline" size="20" />
                  </div>
                </div>
              </template>

              <template v-else-if="msg.isVideo && msg.url">
                <div class="video-wrapper">
                  <video
                    :src="msg.url"
                    class="msg-video"
                    controls
                    playsinline
                  />
                </div>
              </template>

              <template v-else>
                <div class="file-card" :class="{ 'too-big': msg.tooBig }">
                  <div
                    class="file-icon-wrap"
                    :class="fileColorClass(msg.fileType)"
                  >
                    <Icon :name="fileIcon(msg.fileType)" size="20" />
                  </div>
                  <div class="file-info">
                    <span class="file-name">{{ msg.fileName }}</span>
                    <span class="file-size">{{
                      formatSize(msg.fileSize)
                    }}</span>
                    <span v-if="msg.tooBig" class="file-too-big">{{
                      $t("fileTooBig")
                    }}</span>
                  </div>
                  <a
                    v-if="msg.url && !msg.tooBig"
                    :href="msg.url"
                    :download="msg.fileName"
                    class="file-dl-btn"
                  >
                    <Icon name="mdi:download" size="16" />
                  </a>
                </div>
              </template>
            </div>
          </div>
        </div>
      </template>
    </div>

    <!-- Input bar -->
    <div class="chat-input-bar">
      <button class="attach-btn" @click="triggerFileInput">
        <Icon name="mdi:paperclip" size="20" />
        <input
          ref="fileInputEl"
          type="file"
          accept="image/*,video/*,application/pdf,.doc,.docx,.xls,.xlsx,.zip,.txt"
          style="display: none"
          @change="onFileSelected"
        />
      </button>

      <div class="input-wrap">
        <input
          v-model="inputText"
          class="chat-input"
          type="text"
          :placeholder="$t('typeMessage')"
          @keyup.enter="sendText"
        />
      </div>

      <button
        class="send-btn"
        :class="{ active: inputText.trim() }"
        :disabled="!inputText.trim()"
        @click="sendText"
      >
        <Icon name="mdi:send" size="18" />
      </button>
    </div>

    <!-- Lightbox -->
    <Transition name="lb">
      <div v-if="lightboxUrl" class="lightbox" @click="lightboxUrl = null">
        <img :src="lightboxUrl" class="lb-img" @click.stop />
        <button class="lb-close" @click="lightboxUrl = null">
          <Icon name="mdi:close" size="18" />
        </button>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, watch, nextTick } from "vue";

const props = defineProps({
  room: { type: Object, required: true },
});

const { $toast } = useNuxtApp();
const { t } = useI18n();

const messagesEl = ref(null);
const fileInputEl = ref(null);
const inputText = ref("");
const lightboxUrl = ref(null);

const messages = computed(() => props.room.messages.value);

watch(
  () => messages.value.length,
  async () => {
    await nextTick();
    if (messagesEl.value) {
      messagesEl.value.scrollTop = messagesEl.value.scrollHeight;
    }
  },
);

const sendText = () => {
  if (!inputText.value.trim()) return;
  props.room.sendChat(inputText.value.trim());
  inputText.value = "";
};

const triggerFileInput = () => fileInputEl.value?.click();

const onFileSelected = async (e) => {
  const file = e.target.files?.[0];
  if (!file) return;
  await props.room.sendFile(file);
  e.target.value = "";
};

const openImage = (url) => (lightboxUrl.value = url);

const formatTime = (ts) => {
  if (!ts) return "";
  return new Date(ts).toLocaleTimeString([], {
    hour: "2-digit",
    minute: "2-digit",
  });
};

const formatSize = (bytes) => {
  if (!bytes) return "";
  if (bytes < 1024) return bytes + " B";
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + " KB";
  return (bytes / (1024 * 1024)).toFixed(1) + " MB";
};

const fileIcon = (type) => {
  if (!type) return "mdi:file-outline";
  if (type.includes("pdf")) return "mdi:file-pdf-box";
  if (type.includes("word") || type.includes("doc")) return "mdi:file-word-box";
  if (type.includes("sheet") || type.includes("excel") || type.includes("xls"))
    return "mdi:file-excel-box";
  if (type.includes("zip") || type.includes("compressed"))
    return "mdi:folder-zip-outline";
  if (type.includes("text")) return "mdi:file-document-outline";
  return "mdi:file-outline";
};

const fileColorClass = (type) => {
  if (!type) return "fc-default";
  if (type.includes("pdf")) return "fc-pdf";
  if (type.includes("word") || type.includes("doc")) return "fc-word";
  if (type.includes("sheet") || type.includes("excel") || type.includes("xls"))
    return "fc-excel";
  if (type.includes("zip")) return "fc-zip";
  return "fc-default";
};
</script>

<style scoped lang="scss">
/* ── Wrap ────────────────────────────────────────────────── */
.chat-wrap {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
  background: var(--bg-page);
  height: calc(100dvh - 62px);
  padding-bottom: env(safe-area-inset-bottom, 0px);

  @media (max-width: 991px) {
    height: calc(100dvh - 54px);
  }
}

/* ── Messages list ───────────────────────────────────────── */
.messages-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px 14px 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  scroll-behavior: smooth;
  overscroll-behavior: contain;
  -webkit-overflow-scrolling: touch;

  scrollbar-width: thin;
  scrollbar-color: var(--border-color) transparent;

  &::-webkit-scrollbar {
    width: 3px;
  }
  &::-webkit-scrollbar-thumb {
    background: var(--border-color);
    border-radius: 99px;
  }
}

/* ── Empty state ─────────────────────────────────────────── */
.chat-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex: 1;
  gap: 8px;
  padding: 60px 20px;
}

.empty-icon-wrap {
  width: 60px;
  height: 60px;
  border-radius: 20px;
  background: var(--primary-soft);
  border: 1.5px solid var(--primary-mid);
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--primary);
  margin-bottom: 4px;
}

.empty-title {
  margin: 0;
  font-size: 0.88rem;
  font-weight: 600;
  color: var(--text-muted);
}

.empty-sub {
  margin: 0;
  font-size: 0.75rem;
  color: var(--text-muted);
  opacity: 0.55;
}

/* ── Message row ─────────────────────────────────────────── */
.msg-row {
  display: flex;
  align-items: flex-end;
  gap: 8px;
  max-width: 80%;
  align-self: flex-start;
  animation: msgIn 0.2s ease both;

  &.mine {
    flex-direction: row-reverse;
    align-self: flex-end;
  }

  & + .msg-row:not(.mine),
  &.mine + .mine {
    margin-top: 2px;
  }

  &.mine + .msg-row:not(.mine),
  &:not(.mine) + .mine {
    margin-top: 10px;
  }
}

@keyframes msgIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ── Avatar ──────────────────────────────────────────────── */
.msg-avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: var(--secondary);
  color: #fff;
  font-size: 0.65rem;
  font-weight: 800;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;

  &.me {
    background: var(--primary);
  }
}

/* ── Content ─────────────────────────────────────────────── */
.msg-content {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  max-width: 100%;
}

/* ── Meta ────────────────────────────────────────────────── */
.msg-meta {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 0 4px;

  &.meta-mine {
    flex-direction: row-reverse;
  }
}

.msg-author {
  font-size: 0.65rem;
  font-weight: 700;
  color: var(--text-muted);
  letter-spacing: 0.01em;
}

.msg-time {
  font-size: 0.6rem;
  color: var(--text-muted);
  opacity: 0.55;
}

/* ── Bubble ──────────────────────────────────────────────── */
.msg-bubble {
  position: relative;
  background: var(--bg-surface);
  border: 1.5px solid var(--border-color);
  border-radius: 18px 18px 18px 4px;
  padding: 10px 14px;
  font-size: 0.875rem;
  color: var(--text-primary);
  word-break: break-word;
  line-height: 1.55;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);

  &.bubble-mine {
    background: var(--primary-soft);
    border-color: var(--primary-mid);
    border-radius: 18px 18px 4px 18px;
    color: var(--text-primary);
  }
}

.bubble-tick {
  display: inline-flex;
  align-items: center;
  margin-left: 5px;
  vertical-align: middle;
  color: var(--primary);
  opacity: 0.8;
}

/* ── File wrapper ────────────────────────────────────────── */
.msg-file {
  display: flex;
  flex-direction: column;
  gap: 0;
  max-width: 260px;

  @media (min-width: 480px) {
    max-width: 300px;
  }
}

/* ── Image ───────────────────────────────────────────────── */
.img-wrapper {
  position: relative;
  display: inline-block;
  border-radius: 14px;
  overflow: hidden;
  cursor: pointer;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);

  &:hover .img-overlay {
    opacity: 1;
  }
}

.msg-image {
  width: 100%;
  max-width: 220px;
  display: block;
  object-fit: cover;
  border-radius: 14px;
  transition: transform 0.2s;

  @media (min-width: 480px) {
    max-width: 260px;
  }
  .img-wrapper:hover & {
    transform: scale(1.02);
  }
}

.img-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  opacity: 0;
  transition: opacity 0.2s;
  border-radius: 14px;
}

/* ── Video ───────────────────────────────────────────────── */
.video-wrapper {
  border-radius: 14px;
  overflow: hidden;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.msg-video {
  width: 100%;
  max-width: 260px;
  display: block;
  background: #000;
}

/* ── File card ───────────────────────────────────────────── */
.file-card {
  display: flex;
  align-items: center;
  gap: 10px;
  background: var(--bg-surface);
  border: 1.5px solid var(--border-color);
  border-radius: 14px;
  padding: 10px 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
  transition: border-color 0.15s;

  &:active {
    border-color: var(--primary-mid);
  }

  &.too-big {
    border-color: rgba(239, 68, 68, 0.3);
    background: rgba(239, 68, 68, 0.05);
  }
}

.file-icon-wrap {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;

  &.fc-pdf {
    background: rgba(239, 68, 68, 0.12);
    color: #ef4444;
  }
  &.fc-word {
    background: rgba(59, 130, 246, 0.12);
    color: #3b82f6;
  }
  &.fc-excel {
    background: rgba(34, 197, 94, 0.12);
    color: #22c55e;
  }
  &.fc-zip {
    background: rgba(234, 179, 8, 0.12);
    color: #eab308;
  }
  &.fc-default {
    background: var(--bg-elevated);
    color: var(--text-muted);
  }
}

.file-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.file-name {
  font-size: 0.78rem;
  font-weight: 600;
  color: var(--text-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-size {
  font-size: 0.65rem;
  color: var(--text-muted);
}

.file-too-big {
  font-size: 0.62rem;
  color: #ef4444;
  font-weight: 600;
}

.file-dl-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--primary-soft);
  color: var(--primary);
  border: 1.5px solid var(--primary-mid);
  display: flex;
  align-items: center;
  justify-content: center;
  text-decoration: none;
  flex-shrink: 0;
  transition:
    background 0.15s,
    transform 0.1s;

  &:hover {
    background: var(--primary-mid);
  }
  &:active {
    transform: scale(0.93);
  }
}

/* ── Input bar ───────────────────────────────────────────── */
.chat-input-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  padding-bottom: calc(10px + env(safe-area-inset-bottom, 0px));
  background: var(--bg-surface);
  border-top: 1.5px solid var(--border-color);
  flex-shrink: 0;
}

.attach-btn {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  border: 1.5px solid var(--border-color);
  background: var(--bg-elevated);
  color: var(--text-muted);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.15s;
  -webkit-tap-highlight-color: transparent;

  &:hover,
  &:active {
    color: var(--primary);
    border-color: var(--primary-mid);
    background: var(--primary-soft);
  }
}

.input-wrap {
  flex: 1;
  min-width: 0;
}

.chat-input {
  width: 100%;
  height: 42px;
  background: var(--bg-elevated);
  border: 1.5px solid var(--border-color);
  border-radius: 21px;
  padding: 0 16px;
  font-size: 0.875rem;
  color: var(--text-primary);
  font-family: "Tajawal", system-ui, sans-serif;
  outline: none;
  transition:
    border-color 0.15s,
    box-shadow 0.15s;
  box-sizing: border-box;
  -webkit-appearance: none;

  &::placeholder {
    color: var(--text-muted);
  }

  &:focus {
    border-color: var(--primary);
    box-shadow: 0 0 0 3px var(--primary-soft);
  }
}

.send-btn {
  width: 42px;
  height: 42px;
  border-radius: 50%;
  background: var(--bg-elevated);
  border: 1.5px solid var(--border-color);
  color: var(--text-muted);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.2s cubic-bezier(0.34, 1.56, 0.64, 1);
  -webkit-tap-highlight-color: transparent;

  &.active {
    background: var(--primary);
    border-color: var(--primary);
    color: #fff;
    box-shadow: 0 4px 14px var(--primary-mid);
    transform: scale(1.05);
  }

  &:disabled {
    cursor: not-allowed;
    transform: none;
  }

  &.active:active {
    transform: scale(0.93);
  }
}

/* ── Lightbox ────────────────────────────────────────────── */
.lightbox {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.88);
  z-index: 99999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  cursor: pointer;
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
}

.lb-img {
  max-width: min(90vw, 700px);
  max-height: 88vh;
  border-radius: 12px;
  object-fit: contain;
  cursor: default;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.6);
}

.lb-close {
  position: fixed;
  top: max(16px, env(safe-area-inset-top, 16px));
  right: 16px;
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: #fff;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  transition: background 0.15s;

  &:hover {
    background: rgba(255, 255, 255, 0.2);
  }
}

/* ── Lightbox transition ─────────────────────────────────── */
.lb-enter-active,
.lb-leave-active {
  transition: opacity 0.2s ease;
}
.lb-enter-from,
.lb-leave-to {
  opacity: 0;
}

/* ── Responsive ──────────────────────────────────────────── */
@media (max-width: 480px) {
  .msg-row {
    max-width: 88%;
  }

  .messages-list {
    padding: 12px 10px 6px;
    gap: 3px;
  }

  .chat-input-bar {
    padding: 8px 10px;
    padding-bottom: calc(8px + env(safe-area-inset-bottom, 0px));
    gap: 6px;
  }

  .chat-input {
    height: 40px;
    font-size: 16px; /* prevents iOS zoom */
  }

  .attach-btn,
  .send-btn {
    width: 40px;
    height: 40px;
  }

  .msg-bubble {
    font-size: 0.9rem;
    padding: 9px 13px;
  }
}

/* ── RTL send icon flip ──────────────────────────────────── */
.bodyAR .send-btn {
  transform: rotate(180deg);

  &.active {
    transform: rotate(180deg) scale(1.05);
  }
  &.active:active {
    transform: rotate(180deg) scale(0.93);
  }
}
</style>
