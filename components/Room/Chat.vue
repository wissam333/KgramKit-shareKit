<template>
  <div class="chat-wrap">
    <!-- Messages -->
    <div
      ref="messagesEl"
      class="messages-list"
      :dir="$i18n.locale === 'ar' ? 'ltr' : 'rtl'"
    >
      <div v-if="!messages.length" class="chat-empty">
        <Icon name="mdi:chat-outline" size="36" />
        <p>{{ $t("noChatYet") }}</p>
      </div>

      <div
        v-for="msg in messages"
        :key="msg.id"
        class="msg-row"
        :class="{ mine: msg.fromMe }"
      >
        <!-- Avatar -->
        <div class="msg-avatar" :class="{ me: msg.fromMe }">
          {{ (msg.name || "?").charAt(0).toUpperCase() }}
        </div>

        <div class="msg-content">
          <div class="msg-meta">
            <span class="msg-author">{{
              msg.fromMe ? $t("you") : msg.name
            }}</span>
            <span class="msg-time">{{ formatTime(msg.time) }}</span>
          </div>

          <!-- Text message -->
          <div v-if="msg.type === 'chat'" class="msg-bubble">
            {{ msg.text }}
          </div>

          <!-- File / Image / Video -->
          <div v-else-if="msg.type === 'file'" class="msg-file">
            <!-- Image preview -->
            <template v-if="msg.isImage && msg.url">
              <img
                :src="msg.url"
                :alt="msg.fileName"
                class="msg-image"
                @click="openImage(msg.url)"
              />
            </template>

            <!-- Video preview -->
            <template v-else-if="msg.isVideo && msg.url">
              <video :src="msg.url" class="msg-video" controls playsinline />
            </template>

            <!-- Generic file -->
            <template v-else>
              <div class="file-card" :class="{ 'too-big': msg.tooBig }">
                <div class="file-icon">
                  <Icon :name="fileIcon(msg.fileType)" size="22" />
                </div>
                <div class="file-info">
                  <span class="file-name">{{ msg.fileName }}</span>
                  <span class="file-size">{{ formatSize(msg.fileSize) }}</span>
                  <span v-if="msg.tooBig" class="file-too-big">
                    {{ $t("fileTooBig") }}
                  </span>
                </div>
                <a
                  v-if="msg.url && !msg.tooBig"
                  :href="msg.url"
                  :download="msg.fileName"
                  class="file-dl-btn"
                >
                  <Icon name="mdi:download" size="14" />
                </a>
              </div>
            </template>

            <!-- File caption -->
            <div class="file-caption">
              <Icon name="mdi:paperclip" size="11" />
              {{ msg.fileName }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Input bar -->
    <div class="chat-input-bar">
      <!-- Attachment button -->
      <button class="attach-btn" @click="triggerFileInput">
        <Icon name="mdi:paperclip" size="18" />
        <input
          ref="fileInputEl"
          type="file"
          accept="image/*,video/*,application/pdf,.doc,.docx,.xls,.xlsx,.zip,.txt"
          style="display: none"
          @change="onFileSelected"
        />
      </button>

      <!-- Text input -->
      <input
        v-model="inputText"
        class="chat-input"
        type="text"
        :placeholder="$t('typeMessage')"
        @keyup.enter="sendText"
      />

      <!-- Send button -->
      <button class="send-btn" :disabled="!inputText.trim()" @click="sendText">
        <Icon name="mdi:send" size="16" />
      </button>
    </div>

    <!-- Image lightbox -->
    <div v-if="lightboxUrl" class="lightbox" @click="lightboxUrl = null">
      <img :src="lightboxUrl" class="lb-img" @click.stop />
      <button class="lb-close" @click="lightboxUrl = null">
        <Icon name="mdi:close" size="20" />
      </button>
    </div>
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

// Auto-scroll to bottom on new messages
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

const openImage = (url) => {
  lightboxUrl.value = url;
};

const formatTime = (ts) => {
  if (!ts) return "";
  const d = new Date(ts);
  return d.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
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
</script>

<style scoped lang="scss">
.chat-wrap {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* ── Messages ─────────────────────────────────────────────── */
.messages-list {
  flex: 1;
  overflow-y: auto;
  padding: 14px;
  display: flex;
  flex-direction: column;
  gap: 14px;
  scroll-behavior: smooth;

  scrollbar-width: thin;
  scrollbar-color: var(--border-color) transparent;

  &::-webkit-scrollbar {
    width: 4px;
  }
  &::-webkit-scrollbar-thumb {
    background: var(--border-color);
    border-radius: 99px;
  }
}

.chat-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex: 1;
  gap: 10px;
  color: var(--text-muted);
  font-size: 0.82rem;
  padding: 40px 20px;

  p {
    margin: 0;
  }
}

/* ── Message row ─────────────────────────────────────────── */
.msg-row {
  display: flex;
  gap: 8px;
  max-width: 85%;

  &.mine {
    flex-direction: row-reverse;
    align-self: flex-end;

    .msg-meta {
      flex-direction: row-reverse;
    }
  }
}

.msg-avatar {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: #fff;
  font-size: 0.72rem;
  font-weight: 800;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 4px;

  &.me {
    background: linear-gradient(135deg, #14b8a6, #0d9488);
  }
}

.msg-content {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.msg-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.68rem;
}

.msg-author {
  font-weight: 700;
  color: var(--text-muted);
}

.msg-time {
  color: var(--text-muted);
  opacity: 0.6;
}

/* ── Bubble ──────────────────────────────────────────────── */
.msg-bubble {
  background: var(--bg-elevated);
  border: 1.5px solid var(--border-color);
  border-radius: 12px 12px 12px 4px;
  padding: 8px 12px;
  font-size: 0.83rem;
  color: var(--text-primary);
  word-break: break-word;
  line-height: 1.5;

  .mine & {
    background: rgba(20, 184, 166, 0.12);
    border-color: rgba(20, 184, 166, 0.25);
    border-radius: 12px 12px 4px 12px;
  }
}

/* ── File ────────────────────────────────────────────────── */
.msg-file {
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-width: 280px;
}

.msg-image {
  width: 100%;
  max-width: 240px;
  border-radius: 10px;
  cursor: pointer;
  display: block;
  object-fit: cover;
  transition: opacity 0.15s;

  &:hover {
    opacity: 0.88;
  }
}

.msg-video {
  width: 100%;
  max-width: 260px;
  border-radius: 10px;
  display: block;
  background: #000;
}

.file-card {
  display: flex;
  align-items: center;
  gap: 10px;
  background: var(--bg-elevated);
  border: 1.5px solid var(--border-color);
  border-radius: 10px;
  padding: 10px 12px;

  &.too-big {
    border-color: rgba(239, 68, 68, 0.3);
    background: rgba(239, 68, 68, 0.06);
  }
}

.file-icon {
  color: var(--text-muted);
  flex-shrink: 0;
}

.file-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.file-name {
  font-size: 0.76rem;
  font-weight: 700;
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
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: rgba(20, 184, 166, 0.1);
  color: #14b8a6;
  border: 1px solid rgba(20, 184, 166, 0.2);
  display: flex;
  align-items: center;
  justify-content: center;
  text-decoration: none;
  flex-shrink: 0;
  transition: background 0.15s;

  &:hover {
    background: rgba(20, 184, 166, 0.18);
  }
}

.file-caption {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 0.62rem;
  color: var(--text-muted);
}

/* ── Input bar ───────────────────────────────────────────── */
.chat-input-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-top: 1.5px solid var(--border-color);
  background: var(--bg-surface);
  flex-shrink: 0;
}

.attach-btn {
  width: 36px;
  height: 36px;
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

  &:hover {
    color: #14b8a6;
    border-color: rgba(20, 184, 166, 0.3);
    background: rgba(20, 184, 166, 0.08);
  }
}

.chat-input {
  flex: 1;
  height: 38px;
  background: var(--bg-elevated);
  border: 1.5px solid var(--border-color);
  border-radius: 20px;
  padding: 0 14px;
  font-size: 0.83rem;
  color: var(--text-primary);
  font-family: "Tajawal", sans-serif;
  outline: none;
  transition: border-color 0.15s;

  &::placeholder {
    color: var(--text-muted);
    opacity: 0.6;
  }

  &:focus {
    border-color: rgba(20, 184, 166, 0.4);
  }
}

.send-btn {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: #14b8a6;
  border: none;
  color: #fff;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.15s;

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }

  &:hover:not(:disabled) {
    background: #0d9488;
  }
}

/* ── Lightbox ─────────────────────────────────────────────── */
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
}

.lb-img {
  max-width: 90vw;
  max-height: 90vh;
  border-radius: 10px;
  object-fit: contain;
  cursor: default;
}

.lb-close {
  position: fixed;
  top: 16px;
  right: 16px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.15);
  border: none;
  color: #fff;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(6px);
}
</style>
