<template>
  <div class="ss-wrap">
    <!-- Header -->
    <div class="ss-header">
      <div class="ss-header-left">
        <Icon name="mdi:monitor-share" size="16" />
        <span>{{ $t("screenShare") }}</span>
      </div>
      <div class="ss-header-right">
        <div v-if="isSharingLocally" class="live-badge">
          <span class="live-dot" />{{ $t("youAreSharing") }}
        </div>
        <div v-else-if="remoteSharerId" class="live-badge purple">
          <span class="live-dot purple" />
          {{ remoteSharerName }} {{ $t("isSharing") }}
        </div>
      </div>
    </div>

    <!-- Main screen view -->
    <div class="ss-main">
      <!-- Remote share view -->
      <div v-if="remoteSharerId && remoteStream" class="ss-remote-view">
        <video ref="remoteVideoEl" class="ss-video" autoplay playsinline />
        <div class="ss-remote-overlay">
          <button class="ss-overlay-btn" @click="goFullscreen">
            <Icon name="mdi:fullscreen" size="16" />
            {{ $t("fullscreen") }}
          </button>
        </div>
      </div>

      <!-- Local share preview -->
      <div v-else-if="isSharingLocally && localStream" class="ss-local-view">
        <video
          ref="localVideoEl"
          class="ss-video mirror-none"
          autoplay
          muted
          playsinline
        />
        <div class="ss-local-label">
          <Icon name="mdi:eye-outline" size="13" />
          {{ $t("yourScreen") }} — {{ $t("othersCanSee") }}
        </div>
      </div>

      <!-- Empty state -->
      <div v-else class="ss-empty">
        <div class="ss-empty-icon">
          <Icon name="mdi:monitor-off" size="48" />
        </div>
        <h3>{{ $t("noScreenSharing") }}</h3>
        <p>{{ $t("noScreenSharingDesc") }}</p>

        <div class="ss-actions">
          <SharedUiButtonBase
            variant="primary"
            icon-left="mdi:monitor-share"
            :loading="starting"
            @click="startShare"
          >
            {{ $t("shareYourScreen") }}
          </SharedUiButtonBase>

          <div v-if="isMobile" class="mobile-hint">
            <Icon name="mdi:information-outline" size="13" />
            {{ $t("mobileShareHint") }}
          </div>
        </div>
      </div>
    </div>

    <!-- Bottom bar -->
    <div class="ss-footer">
      <div class="footer-left">
        <!-- Who can share list -->
        <div class="sharing-members">
          <div
            v-for="m in allMembers"
            :key="m.peerId"
            class="sharing-member"
            :class="{
              active:
                m.peerId === remoteSharerId || (m.isMe && isSharingLocally),
            }"
          >
            <div class="sm-avatar">
              {{ m.name.charAt(0).toUpperCase() }}
            </div>
            <span class="sm-name">{{ m.isMe ? $t("you") : m.name }}</span>
            <Icon
              v-if="m.peerId === remoteSharerId || (m.isMe && isSharingLocally)"
              name="mdi:monitor-share"
              size="12"
              style="color: #14b8a6"
            />
          </div>
        </div>
      </div>

      <div class="footer-right">
        <SharedUiButtonBase
          v-if="!isSharingLocally"
          variant="primary"
          size="sm"
          icon-left="mdi:monitor-share"
          :loading="starting"
          @click="startShare"
        >
          {{ $t("shareScreen") }}
        </SharedUiButtonBase>

        <SharedUiButtonBase
          v-else
          variant="error"
          size="sm"
          icon-left="mdi:monitor-off"
          @click="stopShare"
        >
          {{ $t("stopSharing") }}
        </SharedUiButtonBase>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";

const props = defineProps({
  room: { type: Object, required: true },
});

const { t } = useI18n();
const { $toast } = useNuxtApp();

const localVideoEl = ref(null);
const remoteVideoEl = ref(null);
const isSharingLocally = ref(false);
const starting = ref(false);
const remoteSharerId = ref(null); // peerId of who is sharing
const remoteStream = ref(null);
let localStream = null;
let screenCall = null;

const isMobile = computed(() => {
  if (!import.meta.client) return false;
  return /Android|iPhone|iPad|iPod|Mobile/i.test(navigator.userAgent);
});

const allMembers = computed(() => [
  {
    peerId: props.room.myPeerId.value,
    name: props.room.myName.value,
    isMe: true,
  },
  ...props.room.members.value,
]);

const remoteSharerName = computed(() => {
  const m = props.room.members.value.find(
    (m) => m.peerId === remoteSharerId.value,
  );
  return m?.name || t("peer");
});

// ── Start screen share ────────────────────────────────────
const startShare = async () => {
  if (starting.value) return;
  starting.value = true;

  try {
    localStream = await props.room.getScreenStream();
    if (!localStream) {
      starting.value = false;
      return;
    }

    isSharingLocally.value = true;

    // Show preview
    await nextTick();
    if (localVideoEl.value) {
      localVideoEl.value.srcObject = localStream;
      localVideoEl.value.play().catch(() => {});
    }

    // Notify room via data channel
    props.room.broadcast({
      type: "screen-share-start",
      peerId: props.room.myPeerId.value,
      name: props.room.myName.value,
    });

    // Call all members with screen stream
    const peer = props.room.getPeer();
    for (const m of props.room.members.value) {
      try {
        const call = peer.call(m.peerId, localStream, {
          metadata: { type: "screen-share", name: props.room.myName.value },
        });
        screenCall = call;
        call.on("close", () => {
          if (isSharingLocally.value) stopShare();
        });
      } catch {}
    }

    // Handle stream end (user pressed browser stop button)
    localStream.getVideoTracks()[0].addEventListener("ended", () => {
      stopShare();
    });
  } catch (e) {
    $toast?.error(t("screenShareFailed"));
  } finally {
    starting.value = false;
  }
};

const stopShare = () => {
  localStream?.getTracks().forEach((t) => t.stop());
  localStream = null;
  screenCall?.close();
  screenCall = null;
  isSharingLocally.value = false;

  props.room.broadcast({
    type: "screen-share-stop",
    peerId: props.room.myPeerId.value,
  });

  if (localVideoEl.value) localVideoEl.value.srcObject = null;
};

const goFullscreen = () => {
  remoteVideoEl.value?.requestFullscreen?.();
};

// ── Listen to room events ─────────────────────────────────
let unsubscribe;
onMounted(() => {
  unsubscribe = props.room.onMessage((data, fromPeerId) => {
    if (data.type === "screen-share-start") {
      remoteSharerId.value = data.peerId;
    } else if (data.type === "screen-share-stop") {
      if (remoteSharerId.value === data.peerId) {
        remoteSharerId.value = null;
        remoteStream.value = null;
        if (remoteVideoEl.value) remoteVideoEl.value.srcObject = null;
      }
    } else if (data.type === "incoming-call") {
      // Answer screen share calls automatically
      if (data.metadata?.type === "screen-share") {
        remoteSharerId.value = data.peerId;
        props.room.getPeer().call(data.peerId, new MediaStream()).close();
        data.call.answer(new MediaStream());
        data.call.on("stream", (stream) => {
          remoteStream.value = stream;
          nextTick(() => {
            if (remoteVideoEl.value) {
              remoteVideoEl.value.srcObject = stream;
              remoteVideoEl.value.play().catch(() => {});
            }
          });
        });
        data.call.on("close", () => {
          remoteSharerId.value = null;
          remoteStream.value = null;
        });
      }
    } else if (data.type === "remote-stream") {
      // Capture screen share streams (media calls)
      // Screen share calls come from PeerJS call events
    }
  });

  // Override peer's call handler to intercept screen share
  const peer = props.room.getPeer();
  if (peer) {
    peer.on("call", (call) => {
      if (call.metadata?.type === "screen-share") {
        remoteSharerId.value = call.peer;
        call.answer(new MediaStream());
        call.on("stream", (stream) => {
          remoteStream.value = stream;
          nextTick(() => {
            if (remoteVideoEl.value) {
              remoteVideoEl.value.srcObject = stream;
              remoteVideoEl.value.play().catch(() => {});
            }
          });
        });
        call.on("close", () => {
          if (remoteSharerId.value === call.peer) {
            remoteSharerId.value = null;
            remoteStream.value = null;
          }
        });
      }
    });
  }
});

onUnmounted(() => {
  unsubscribe?.();
  if (isSharingLocally.value) stopShare();
});
</script>

<style scoped lang="scss">
.ss-wrap {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* ── Header ──────────────────────────────────────────────── */
.ss-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  border-bottom: 1.5px solid var(--border-color);
  background: var(--bg-surface);
  flex-shrink: 0;
  gap: 10px;
}

.ss-header-left {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.82rem;
  font-weight: 700;
  color: var(--text-primary);
}

.ss-header-right {
  display: flex;
  align-items: center;
  gap: 8px;
}

.live-badge {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.68rem;
  font-weight: 800;
  color: #22c55e;
  background: rgba(34, 197, 94, 0.1);
  border: 1px solid rgba(34, 197, 94, 0.2);
  padding: 3px 9px;
  border-radius: 20px;

  &.purple {
    color: #6366f1;
    background: rgba(99, 102, 241, 0.1);
    border-color: rgba(99, 102, 241, 0.2);
  }
}

.live-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #22c55e;
  animation: livePulse 1.5s ease-in-out infinite;

  &.purple {
    background: #6366f1;
  }
}

@keyframes livePulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.4;
  }
}

/* ── Main area ───────────────────────────────────────────── */
.ss-main {
  flex: 1;
  overflow: hidden;
  position: relative;
  background: #0a0f1a;
  display: flex;
  align-items: center;
  justify-content: center;
}

.ss-remote-view,
.ss-local-view {
  width: 100%;
  height: 100%;
  position: relative;
}

.ss-video {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.mirror-none {
  transform: none;
}

.ss-remote-overlay {
  position: absolute;
  bottom: 12px;
  right: 12px;
  display: flex;
  gap: 8px;
}

.ss-overlay-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 13px;
  border-radius: 20px;
  background: rgba(0, 0, 0, 0.65);
  border: 1px solid rgba(255, 255, 255, 0.15);
  color: #fff;
  font-size: 0.75rem;
  font-weight: 700;
  font-family: "Tajawal", sans-serif;
  cursor: pointer;
  backdrop-filter: blur(6px);
  transition: background 0.15s;

  &:hover {
    background: rgba(0, 0, 0, 0.85);
  }
}

.ss-local-label {
  position: absolute;
  bottom: 10px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(20, 184, 166, 0.85);
  color: #fff;
  font-size: 0.72rem;
  font-weight: 700;
  padding: 5px 13px;
  border-radius: 20px;
  backdrop-filter: blur(4px);
  white-space: nowrap;
}

/* ── Empty state ─────────────────────────────────────────── */
.ss-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  padding: 32px 20px;
  text-align: center;
  color: rgba(255, 255, 255, 0.5);
}

.ss-empty-icon {
  opacity: 0.25;
}

.ss-empty h3 {
  font-size: 1rem;
  font-weight: 800;
  color: rgba(255, 255, 255, 0.6);
  margin: 0;
}

.ss-empty p {
  font-size: 0.78rem;
  color: rgba(255, 255, 255, 0.35);
  margin: 0;
  max-width: 280px;
}

.ss-actions {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  margin-top: 4px;
}

.mobile-hint {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.7rem;
  color: rgba(255, 255, 255, 0.3);
}

/* ── Footer ──────────────────────────────────────────────── */
.ss-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  border-top: 1.5px solid var(--border-color);
  background: var(--bg-surface);
  flex-shrink: 0;
  gap: 12px;
}

.footer-left {
  flex: 1;
  min-width: 0;
  overflow: hidden;
}

.footer-right {
  flex-shrink: 0;
}

/* ── Sharing members ─────────────────────────────────────── */
.sharing-members {
  display: flex;
  align-items: center;
  gap: 8px;
  overflow-x: auto;
  scrollbar-width: none;
  padding-bottom: 2px;

  &::-webkit-scrollbar {
    display: none;
  }
}

.sharing-member {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 20px;
  border: 1.5px solid var(--border-color);
  background: var(--bg-elevated);
  flex-shrink: 0;
  transition: all 0.15s;

  &.active {
    border-color: rgba(20, 184, 166, 0.35);
    background: rgba(20, 184, 166, 0.08);
  }
}

.sm-avatar {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: linear-gradient(135deg, #14b8a6, #6366f1);
  color: #fff;
  font-size: 0.6rem;
  font-weight: 800;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.sm-name {
  font-size: 0.72rem;
  font-weight: 700;
  color: var(--text-primary);
  white-space: nowrap;
}
</style>
