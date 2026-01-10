<script setup lang="ts">
import { ref, computed } from "vue";
import { ElNotification, ElMessage } from "element-plus";
import { request } from "@/utils/requests";
import type { BaseMovieInfo } from "@/types/Movie";

export interface ScreenShareConfig {
  includeAudio: boolean;
  quality: 'high' | 'medium' | 'low';
  frameRate: number;
  bitrate: number;
}

const Props = defineProps<{
  token: string;
  roomId: string;
  movieId: string;
  movieInfo: BaseMovieInfo;
}>();

const emit = defineEmits<{
  (e: 'close'): void;
  (e: 'sharing-started'): void;
  (e: 'sharing-stopped'): void;
}>();

const open = ref(false);
const sharing = ref(false);
const connecting = ref(false);
const stream = ref<MediaStream | null>(null);
const mediaRecorder = ref<MediaRecorder | null>(null);

const qualityLabels = {
  high: '高清 (1080p@30fps)',
  medium: '标清 (720p@15fps)',
  low: '流畅 (480p@15fps)'
};

const config = computed((): ScreenShareConfig => {
  return Props.movieInfo.vendorInfo?.screenShareConfig || {
    includeAudio: true,
    quality: 'high',
    frameRate: 30,
    bitrate: 2000
  };
});

const openDialog = () => {
  open.value = true;
};

const closeDialog = () => {
  stopSharing();
  open.value = false;
  emit('close');
};

const getVideoConstraints = () => {
  const constraints: any = {
    displaySurface: 'monitor'
  };

  switch (config.value.quality) {
    case 'high':
      constraints.width = { ideal: 1920 };
      constraints.height = { ideal: 1080 };
      constraints.frameRate = { ideal: 30 };
      break;
    case 'medium':
      constraints.width = { ideal: 1280 };
      constraints.height = { ideal: 720 };
      constraints.frameRate = { ideal: 15 };
      break;
    case 'low':
      constraints.width = { ideal: 854 };
      constraints.height = { ideal: 480 };
      constraints.frameRate = { ideal: 15 };
      break;
  }

  return constraints;
};

const sendStreamToServer = async (chunk: Blob) => {
  try {
    await request({
      url: "/api/room/movie/live/screen-share/push/" + Props.movieId,
      method: 'POST',
      data: chunk,
      headers: {
        'Authorization': 'Bearer ' + Props.token,
        'X-Room-Id': Props.roomId,
        'Content-Type': 'video/x-flv'
      }
    });
  } catch (err: any) {
    console.error('推送屏幕共享数据失败:', err);
  }
};

const startSharing = async () => {
  try {
    connecting.value = true;

    stream.value = await navigator.mediaDevices.getDisplayMedia({
      video: getVideoConstraints(),
      audio: config.value.includeAudio
    });

    stream.value.getVideoTracks().forEach(track => {
      track.onended = () => {
        stopSharing();
        ElMessage.info('屏幕共享已结束');
      };
    });

    const mimeTypes = [
      'video/webm;codecs=vp8,opus',
      'video/webm;codecs=vp9,opus',
      'video/webm',
      'video/mp4'
    ];

    let selectedMimeType = '';
    for (const type of mimeTypes) {
      if (MediaRecorder.isTypeSupported(type)) {
        selectedMimeType = type;
        break;
      }
    }

    if (!selectedMimeType) {
      throw new Error('浏览器不支持任何视频编码格式');
    }

    mediaRecorder.value = new MediaRecorder(stream.value, {
      mimeType: selectedMimeType,
      videoBitsPerSecond: config.value.bitrate * 1000
    });

    mediaRecorder.value.ondataavailable = async (event) => {
      if (event.data && event.data.size > 0) {
        await sendStreamToServer(event.data);
      }
    };

    mediaRecorder.value.onstop = () => {
      console.log('MediaRecorder 已停止');
    };

    mediaRecorder.value.start(500);
    sharing.value = true;
    connecting.value = false;

    ElNotification({
      title: '屏幕共享已开始',
      type: 'success',
      message: '数据正在推送到服务器'
    });

    emit('sharing-started');

  } catch (err: any) {
    connecting.value = false;
    console.error('屏幕共享启动失败:', err);
    ElNotification({
      title: '屏幕共享启动失败',
      message: err.message || '未知错误',
      type: 'error'
    });
  }
};

const stopSharing = () => {
  if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
    try {
      mediaRecorder.value.stop();
    } catch (e) {
      console.error('停止 MediaRecorder 失败:', e);
    }
  }

  if (stream.value) {
    stream.value.getTracks().forEach(track => {
      track.stop();
    });
    stream.value = null;
  }

  sharing.value = false;
  mediaRecorder.value = null;
  emit('sharing-stopped');
};

defineExpose({
  openDialog,
  closeDialog,
  startSharing,
  stopSharing
});
</script>

<template>
  <el-dialog
    v-model="open"
    destroy-on-close
    title="屏幕共享"
    class="rounded-lg dark:bg-zinc-800 w-full md:w-3/4"
    @closed="closeDialog"
  >
    <div class="p-4">
      <div class="mb-4 p-3 bg-zinc-100 dark:bg-zinc-700 rounded-lg">
        <h3 class="text-lg font-medium mb-2">共享配置</h3>
        <div class="space-y-2 text-sm">
          <div>
            <span class="text-zinc-600 dark:text-zinc-400">视频质量：</span>
            <span class="font-medium">{{ qualityLabels[config.quality] }}</span>
          </div>
          <div>
            <span class="text-zinc-600 dark:text-zinc-400">音频：</span>
            <span class="font-medium">{{ config.includeAudio ? '包含系统音频' : '无音频' }}</span>
          </div>
          <div>
            <span class="text-zinc-600 dark:text-zinc-400">目标码率：</span>
            <span class="font-medium">{{ config.bitrate }} kbps</span>
          </div>
        </div>
      </div>

      <div class="mb-4 p-3 rounded-lg" :class="sharing ? 'bg-green-100 dark:bg-green-900' : 'bg-zinc-100 dark:bg-zinc-700'">
        <div class="flex items-center">
          <div
            class="w-3 h-3 rounded-full mr-3"
            :class="{
              'bg-green-500': sharing,
              'bg-yellow-500': connecting,
              'bg-gray-400': !sharing && !connecting
            }"
          />
          <span class="font-medium">
            {{ connecting ? '正在连接...' : (sharing ? '正在共享' : '未开始共享') }}
          </span>
        </div>
      </div>

      <div class="mb-4 text-sm text-zinc-600 dark:text-zinc-400">
        <p class="mb-2"><strong>使用说明：</strong></p>
        <ul class="list-disc pl-5 space-y-1">
          <li>点击"开始共享"后，选择要共享的屏幕、窗口或标签页</li>
          <li>共享过程中，选择内容的变化将实时传输到房间</li>
          <li>停止共享可点击"停止共享"按钮或关闭浏览器共享提示</li>
          <li>注意：屏幕共享会消耗较多带宽和 CPU 资源</li>
        </ul>
      </div>

      <div class="p-3 bg-blue-50 dark:bg-blue-900/30 rounded-lg text-sm">
        <p class="text-blue-800 dark:text-blue-200">
          <strong>当前实现：</strong>使用 MediaRecorder 记录屏幕内容（基础版本）
        </p>
        <p class="text-blue-700 dark:text-blue-300 mt-1">
          <strong>优化方案：</strong>将集成 WASM FLV 编码器以降低延迟和提高兼容性
        </p>
      </div>
    </div>

    <template #footer>
      <div class="flex justify-end space-x-3">
        <button
          v-if="!sharing"
          class="btn"
          :disabled="connecting"
          @click="startSharing"
        >
          {{ connecting ? '连接中...' : '开始共享' }}
        </button>
        <button
          v-else
          class="btn btn-error"
          @click="stopSharing"
        >
          停止共享
        </button>
        <button class="btn" @click="closeDialog">关闭</button>
      </div>
    </template>
  </el-dialog>
</template>
