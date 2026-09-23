<template>
  <div class="diagnose">
    <n-card title="一键诊断：为什么一连命令就断" class="mb-4">
      <n-alert type="info" :show-icon="true" class="mb-4">
        选一个角色，点「一键诊断」。页面会自动连上游戏服务器、用你选的
        <b>clientVersion</b> 发一条「获取角色信息」，并把
        <b>连接关闭码（code）</b> 和 <b>服务器返回的错误码</b> 用中文显示出来。
        你不用开 F12，把页面最下面的「结论」发我截图即可。
      </n-alert>

      <n-space align="center" :wrap="true">
        <n-form-item label="选择角色" class="role-form">
          <n-select
            v-model:value="selectedRoleId"
            placeholder="请选择要诊断的角色"
            :options="roleOptions"
            style="width: 260px"
          />
        </n-form-item>
        <n-form-item label="clientVersion" class="ver-form">
          <n-radio-group v-model:value="versionMode">
            <n-space>
              <n-radio value="old">旧版 2.21.2</n-radio>
              <n-radio value="new">新版 2.43.4</n-radio>
              <n-radio value="custom">自定义</n-radio>
            </n-space>
          </n-radio-group>
        </n-form-item>
      </n-space>

      <n-input
        v-if="versionMode === 'custom'"
        v-model:value="customVersion"
        placeholder="粘贴要测试的 clientVersion，例如 2.43.4-a7db1319a3025acb-wx"
        class="mb-3"
      />

      <n-space align="center">
        <n-button
          type="primary"
          :loading="running"
          :disabled="!selectedRoleId"
          @click="diagnose"
        >
          一键诊断
        </n-button>
        <n-text depth="3" class="hint">
          提示：先点「旧版」试一次，再点「新版」试一次，对比结论即可定位是不是版本问题。
        </n-text>
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

const OLD_VER = "2.21.2-fa918e1997301834-wx";
const NEW_VER = "2.43.4-a7db1319a3025acb-wx";

const selectedRoleId = ref(null);
const running = ref(false);
const steps = ref([]);
const closeInfo = ref(null); // { code, reason }
const firstMsg = ref(null); // { cmd, code }
const messagesSeen = ref(0);
const roleTest = ref(null); // { status: 'ok'|'error', code?, desc?, name? }

const versionMode = ref("old"); // 'old' | 'new' | 'custom'
const customVersion = ref("");

const chosenVersion = computed(() => {
  if (versionMode.value === "new") return NEW_VER;
  if (versionMode.value === "custom") return customVersion.value.trim() || OLD_VER;
  return OLD_VER;
});

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
  roleTest.value = null;
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
    if (data instanceof ArrayBuffer) return g_utils.parse(data, "auto");
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

  pushStep(
    "info",
    "开始连接",
    `正在建立 WebSocket 连接…（测试 clientVersion: ${chosenVersion.value}）`,
  );

  tokenStore.createWebSocketConnection(
    selectedRoleId.value,
    token.token,
    token.wsUrl,
  );

  // 等一拍，确保 client 已创建
  await new Promise((r) => setTimeout(r, 400));

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
    runRoleTest(client);
  };
  const onMessage = (evt) => {
    messagesSeen.value += 1;
    const pkt = readFrame(evt.data);
    const cmd = pkt?.cmd || pkt?.c || "?";
    const code = pkt?.code;
    if (!firstMsg.value) {
      firstMsg.value = { cmd, code };
      const desc = code
        ? errorCodeMap[code] || `业务码 ${code}`
        : "成功(无错误码)";
      pushStep(
        "success",
        "收到服务器首条消息",
        `命令: ${cmd} ｜ 返回码: ${code ?? "无"} ｜ ${desc}`,
      );
    }
  };
  const onClose = (evt) => {
    closeInfo.value = { code: evt.code, reason: evt.reason || "" };
    const reasonText = evt.reason
      ? `（原因: ${evt.reason}）`
      : "（无原因文本）";
    let type = "error";
    if (evt.code === 1000) type = "warning";
    pushStep(type, `连接被关闭 code=${evt.code}`, reasonText);
    finish();
  };

  if (socket.readyState === WebSocket.OPEN) onOpen();
  socket.addEventListener("open", onOpen);
  socket.addEventListener("message", onMessage);
  socket.addEventListener("close", onClose);

  // 兜底：8 秒还没结论，给个超时提示
  setTimeout(() => {
    if (running.value && !closeInfo.value && !roleTest.value) {
      pushStep(
        "warning",
        "超时未关闭",
        "连接保持中，但 8 秒内未收到角色信息响应也未断开。可手动断开看结果。",
      );
      running.value = false;
    }
  }, 8000);
};

const runRoleTest = async (client) => {
  // 等 store 自动发的 role_getroleinfo 先出去，再发我们带版本的测试
  await new Promise((r) => setTimeout(r, 600));
  if (!client) return;

  pushStep(
    "info",
    "发送 role_getroleinfo",
    `携带 clientVersion: ${chosenVersion.value}`,
  );

  try {
    const res = await client.getRoleInfo({ clientVersion: chosenVersion.value });
    const name =
      res?.name ||
      res?.roleName ||
      res?.nickName ||
      (res?.role && res.role.name) ||
      null;
    roleTest.value = { status: "ok", name };
    pushStep(
      "success",
      "角色信息成功返回",
      name ? `角色: ${name}` : "拿到了角色数据（无名称字段）",
    );
  } catch (e) {
    const m = e?.message || String(e);
    const mm = m.match(/(\d{4,})/);
    const code = mm ? mm[1] : null;
    roleTest.value = { status: "error", code, desc: m };
    pushStep("error", "角色信息失败", m);
  }
  finish();
};

const finish = () => {
  // 关键证据（关闭码 或 角色测试结果）齐了再给结论
  if (!closeInfo.value && !roleTest.value) return;
  running.value = false;

  const code = closeInfo.value?.code;
  const reason = closeInfo.value?.reason || "";

  // 1. 角色信息成功 → 连接完全正常
  if (roleTest.value?.status === "ok") {
    verdictTitle.value = "连接正常 ✓";
    verdictStatus.value = "success";
    verdict.value =
      `用 clientVersion「${chosenVersion.value}」成功拿到角色信息，说明 token 与连接都正常。\n` +
      `你之前遇到的 200020 / 已断开，不是版本或部署问题，而是：\n` +
      `① 该账号在手机游戏或其他设备在线，把网页连接挤掉；或\n` +
      `② 某些活动/功能当前不在开放时间。\n` +
      `👉 请彻底关闭手机上的咸鱼之王，只留一个 ctt.ccwu.cc 标签页，再点一次。若仍断，把结论发我。`;
    return;
  }

  // 2. 角色信息失败，带了业务码
  if (roleTest.value?.status === "error") {
    const c = roleTest.value.code;
    if (versionMode.value === "old" && c) {
      verdictTitle.value = "疑似 clientVersion 过期";
      verdictStatus.value = "warning";
      verdict.value =
        `角色信息查询返回业务码 ${c}（${errorCodeMap[c] || "未知"}）。\n` +
        `当前测的是旧版 ${OLD_VER}。请上方切到「新版 2.43.4」再点一次诊断：\n` +
        `· 若新版成功 → 确认是版本问题，我直接把代码改成新版推你 fork；\n` +
        `· 若新版也报同样的码或被 1005 踢 → 不是版本问题，是会话被占（关手机游戏）。`;
      return;
    }
    if (versionMode.value === "new" && c) {
      verdictTitle.value = "新版也失败";
      verdictStatus.value = "error";
      verdict.value =
        `即使用最新 clientVersion ${NEW_VER}，角色信息仍返回 ${c} 或连接被关。说明不是版本号问题，而是服务器在拒绝这条会话。\n` +
        `最可能：该账号在手机游戏/其他设备在线，游戏只允许一条连接。\n` +
        `👉 请彻底关闭手机咸鱼之王，再点一次（旧版/新版都行）。`;
      return;
    }
    // 自定义版本
    verdictTitle.value = `角色信息失败（码 ${c || "无"}）`;
    verdictStatus.value = "error";
    verdict.value =
      `用自定义版本「${chosenVersion.value}」仍失败：${roleTest.value.desc}\n` +
      `可再试「旧版」和「新版」对比。若都失败，基本确定是会话被占（关手机游戏）或游戏方封禁。`;
    return;
  }

  // 3. 没拿到角色信息，只有关闭码
  if (code === 1005 || code === 1006) {
    verdictTitle.value = "服务器直接踢连接（最可能：同号互踢）";
    verdictStatus.value = "error";
    verdict.value =
      `连接建立后服务器直接以 ${code} 关闭，且没返回任何角色数据。\n` +
      `这说明服务器在握手后立刻拒掉了这条连接。最常见原因：该账号在别处（手机游戏 / 其他浏览器标签页 / 其他设备）已经在线，游戏服务器只允许一条会话，新连接被挤掉。\n` +
      `👉 请：① 彻底关闭手机上的咸鱼之王（杀后台）；② 只开一个 ctt.ccwu.cc 标签页；③ 重新点「一键诊断」（可试「新版」clientVersion）。\n` +
      `若关掉手机游戏仍 1005，那就是游戏方对第三方工具做了连接层封禁，纯前端改不了。`;
    return;
  }

  if (code === 1000) {
    verdictTitle.value = "前端主动断开";
    verdictStatus.value = "warning";
    verdict.value =
      `关闭码 1000（正常关闭），原因是「${reason || "未知"}」。通常是页面或调度逻辑自己关的，不是游戏服务器踢的。请把 reason 文字发我，小二继续查。`;
    return;
  }

  verdictTitle.value = `连接被关闭 code=${code}`;
  verdictStatus.value = "error";
  verdict.value =
    `关闭码 ${code}，原因「${reason || "无"}」。\n` +
    `连接期间没有收到角色信息。把这段发我，小二据此判断。`;
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
.ver-form {
  margin-bottom: 0;
}
.hint {
  max-width: 360px;
}
.mt-4 {
  margin-top: 16px;
}
.mb-3 {
  margin-bottom: 12px;
}
</style>
