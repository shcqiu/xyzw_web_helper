<template>
  <div class="diagnose">
    <n-card title="一键诊断：为什么一连命令就断" class="mb-4">
      <n-alert type="info" :show-icon="true" class="mb-4">
        选一个角色，点「一键诊断」。页面会自动连上游戏服务器、发一条命令，并把
        <b>连接关闭码（code）</b> 和 <b>服务器返回的错误码</b> 用中文显示出来。
        你不用开 F12、不用看控制台，把页面最下面的「结论」发给我截图即可。
      </n-alert>

      <n-space align="center">
        <n-form-item label="选择角色" class="role-form">
          <n-select
            v-model:value="selectedRoleId"
            placeholder="请选择要诊断的角色"
            :options="roleOptions"
            style="width: 260px"
          />
        </n-form-item>
        <n-button
          type="primary"
          :loading="running"
          :disabled="!selectedRoleId"
          @click="diagnose"
        >
          一键诊断
        </n-button>
      </n-space>

      <!-- 时间线 -->
      <n-card title="诊断过程" size="small" class="mt-4">
        <n-empty v-if="steps.length === 0" description="还没开始，点上面的按钮" />
        <n-timeline v-else>
          <n-timeline-item
            v-for="(s, i) in steps"
            :key="i"
            :type="s.type"
            :title="s.title"
            :content="s.content"
            :time="s.time"
          />
        </n-timeline>
      </n-card>

      <!-- 结论 -->
      <n-card v-if="verdict" title="结论" size="small" class="mt-4">
        <n-result
          :status="verdictStatus"
          :title="verdictTitle"
          :description="verdict"
        >
          <template #footer>
            <n-space justify="center">
              <n-button size="small" @click="copyVerdict">复制结论</n-button>
            </n-space>
          </template>
        </n-result>
      </n-card>
    </n-card>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from "vue";
import { useMessage } from "naive-ui";
import { useTokenStore } from "@stores/tokenStore";
import { g_utils } from "@/utils/bonProtocol.js";

const message = useMessage();
const tokenStore = useTokenStore();

const selectedRoleId = ref(null);
const running = ref(false);
const steps = ref([]);
const closeInfo = ref(null); // { code, reason }
const firstMsg = ref(null); // { cmd, code }
const messagesSeen = ref(0);

const verdict = ref("");
const verdictTitle = ref("");
const verdictStatus = ref("info");

const roleOptions = computed(() =>
  tokenStore.gameTokens.map((t) => ({
    label: `${t.name} (${t.server})`,
    value: t.id,
  })),
);

const now = () => new Date().toLocaleTimeString("zh-CN");

const pushStep = (type, title, content) => {
  steps.value.push({ type, title, content, time: now() });
};

const reset = () => {
  steps.value = [];
  closeInfo.value = null;
  firstMsg.value = null;
  messagesSeen.value = 0;
  verdict.value = "";
  verdictTitle.value = "";
  verdictStatus.value = "info";
};

const errorCodeMap = {
  200020: "出了点小问题，请尝试重启游戏解决～",
  200760: "您当前看到的界面已发生变化，请重新登录",
  200400: "操作太快，请稍后再试",
  4800080: "不在规定时间内或未到报名阶段",
  4800040: "俱乐部没有报名",
  2100010: "活动未开放",
};

const readFrame = (data) => {
  try {
    if (typeof data === "string") return JSON.parse(data);
    if (data instanceof ArrayBuffer)
      return g_utils.parse(data, "auto");
    return null;
  } catch {
    return null;
  }
};

const diagnose = async () => {
  if (!selectedRoleId.value) {
    message.error("请先选择角色");
    return;
  }
  running.value = true;
  reset();

  const token = tokenStore.gameTokens.find((t) => t.id === selectedRoleId.value);
  if (!token) {
    message.error("未找到 Token 数据");
    running.value = false;
    return;
  }

  pushStep("info", "开始连接", "正在建立 WebSocket 连接…");

  // 用 store 建立连接（内部会解析 token、构造 wsUrl、连上后自动发 role_getroleinfo）
  tokenStore.createWebSocketConnection(
    selectedRoleId.value,
    token.token,
    token.wsUrl,
  );

  // 等一拍，确保 client 已创建
  await new Promise((r) => setTimeout(r, 300));

  const conn = tokenStore.wsConnections[selectedRoleId.value];
  const client = conn?.client;
  const socket = client?.socket;

  if (!socket) {
    pushStep("error", "连接未建立", "没有拿到 WebSocket 实例，请重试");
    running.value = false;
    return;
  }

  const onOpen = () => {
    pushStep("success", "已连接", "WebSocket 握手成功（token 有效）");
  };
  const onMessage = (evt) => {
    messagesSeen.value += 1;
    const pkt = readFrame(evt.data);
    const cmd = pkt?.cmd || pkt?.c || "?";
    const code = pkt?.code;
    if (!firstMsg.value) {
      firstMsg.value = { cmd, code };
      const desc = code ? errorCodeMap[code] || `业务码 ${code}` : "成功(无错误码)";
      pushStep(
        "success",
        "收到服务器首条消息",
        `命令: ${cmd} ｜ 返回码: ${code ?? "无"} ｜ ${desc}`,
      );
    }
  };
  const onClose = (evt) => {
    closeInfo.value = { code: evt.code, reason: evt.reason || "" };
    const reasonText = evt.reason ? `（原因: ${evt.reason}）` : "（无原因文本）";
    let type = "error";
    if (evt.code === 1000) type = "warning";
    pushStep(type, `连接被关闭 code=${evt.code}`, reasonText);
    finish();
  };

  if (socket.readyState === WebSocket.OPEN) onOpen();
  socket.addEventListener("open", onOpen);
  socket.addEventListener("message", onMessage);
  socket.addEventListener("close", onClose);

  // 兜底：6 秒还没关闭也没消息，给个超时结论
  setTimeout(() => {
    if (running.value && !closeInfo.value) {
      pushStep(
        "warning",
        "超时未关闭",
        "连接保持中，未观察到自动断开。可手动点断开看结果。",
      );
      running.value = false;
    }
  }, 6000);
};

const finish = () => {
  running.value = false;
  const code = closeInfo.value?.code;
  const reason = closeInfo.value?.reason || "";

  if (code === undefined) {
    verdictTitle.value = "无法判定";
    verdictStatus.value = "info";
    verdict.value = "诊断过程中没有捕获到连接关闭事件，请重试。";
    return;
  }

  if (code === 1000) {
    verdictTitle.value = "前端主动断开";
    verdictStatus.value = "warning";
    verdict.value =
      `关闭码 1000（正常关闭），原因是「${reason || "未知"}」。这通常是页面或调度逻辑自己关的，不是游戏服务器踢的。请把 reason 文字发我，小二继续查。`;
    return;
  }

  // code 1006 = 异常断开，服务器/网络层直接断
  if (code === 1006) {
    if (firstMsg.value && firstMsg.value.code && firstMsg.value.code !== 0) {
      verdictTitle.value = "服务器返回业务错误后断开";
      verdictStatus.value = "error";
      const desc =
        errorCodeMap[firstMsg.value.code] || `业务码 ${firstMsg.value.code}`;
      verdict.value =
        `连接建立后，服务器先回了「${firstMsg.value.cmd} 错误码 ${firstMsg.value.code}（${desc}）」，随后以 1006 断开。\n` +
        `这说明包能被服务器解析（编解码没问题），但这次会话被拒绝了。最常见原因是：该账号在别处（手机游戏 / 其他浏览器标签页 / 其他设备）已经在线，把这条连接挤掉了；也可能是 token 导入方式生成的会话被游戏端占用。\n` +
        `👉 请先把手机上的咸鱼之王彻底关掉，并只保留一个 ctt.ccwu.cc 的浏览器标签页，再点一次「一键诊断」。`;
    } else {
      verdictTitle.value = "服务器直接踢连接（最可能：同号互踢）";
      verdictStatus.value = "error";
      verdict.value =
        `关闭码 1006、无原因文本 —— 服务器在收到第一条命令后直接把连接RST掉了。\n` +
        `最可能是：这个账号在别处已经在线（手机游戏、其他标签页、其他设备），游戏服务器只允许一条会话，新连接一发包就被判定冲突而踢掉。\n` +
        `👉 请：① 彻底关闭手机上的咸鱼之王（杀后台）；② 只开一个 ctt.ccwu.cc 标签页；③ 重新点「一键诊断」。\n` +
        `如果关掉手机游戏和一个标签页后还断，那就不是互踢，请把这次的「诊断过程」发我，小二再查协议/版本号。`;
    }
    return;
  }

  // 其他关闭码
  verdictTitle.value = `连接被关闭 code=${code}`;
  verdictStatus.value = "error";
  verdict.value =
    `关闭码 ${code}，原因「${reason || "无"}」。\n` +
    (firstMsg.value
      ? `连接期间首条消息: ${firstMsg.value.cmd} 返回码 ${firstMsg.value.code ?? "无"}。`
      : "连接期间没有收到任何服务器消息。") +
    `\n把这段发我，小二据此判断。`;
};

const copyVerdict = () => {
  const text =
    `【连接诊断结论】\n` +
    steps.value.map((s) => `- ${s.title}: ${s.content}`).join("\n") +
    `\n结论: ${verdict.value}`;
  navigator.clipboard?.writeText(text).then(
    () => message.success("已复制"),
    () => message.warning("复制失败，请手动截图"),
  );
};

onUnmounted(() => {
  // 诊断页卸载时，若还在跑，断开当前角色连接，避免留尾巴
  if (selectedRoleId.value) {
    tokenStore.closeWebSocketConnection(selectedRoleId.value);
  }
});
</script>

<style scoped>
.diagnose {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}
.role-form {
  margin-bottom: 0;
}
.mt-4 {
  margin-top: 16px;
}
</style>
