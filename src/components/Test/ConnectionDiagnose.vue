<template>
  <div class="diagnose">
    <n-card title="一键诊断：为什么一连命令就断" class="mb-4">
      <n-alert type="info" :show-icon="true" class="mb-4">
        选一个角色，点「一键诊断」。页面会<b>直接</b>连游戏服务器，并依次发 4 条命令
        （角色信息 / 活动 / 数据包版本 / 爬塔信息），把每条的返回码和连接关闭码（code）用中文显示。
        一次点就能看出是「整条会话被拒」还是「只有某个命令被拒」。不用开 F12。
        <br />
        <b>clientVersion</b> 已对齐作者最新版 2.43.4（前两次诊断证明版本号不是根因），这里切换仅供对比。
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
              <n-radio value="new">新版 2.43.4</n-radio>
              <n-radio value="old">旧版 2.21.2</n-radio>
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
        <n-button :disabled="running" @click="reset">清空</n-button>
        <n-text depth="3" class="hint">
          若结论指向「手机游戏在线」，关掉手机咸鱼之王后点这里重测。
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
        <n-result :status="verdictStatus" :title="verdictTitle" :description="verdict">
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
import { XyzwWebSocketClient } from "@/utils/xyzwWebSocket.js";
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
const results = ref({}); // { role:{status,code,desc}, activity:..., databundle:..., tower:... }
const myClient = ref(null);

const versionMode = ref("new"); // 'new' | 'old' | 'custom'
const customVersion = ref("");

const chosenVersion = computed(() => {
  if (versionMode.value === "new") return NEW_VER;
  if (versionMode.value === "custom")
    return customVersion.value.trim() || OLD_VER;
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
  results.value = {};
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
  1500010: "已经全部通关",
  1500040: "上座塔的奖励未领取",
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

// 一次发送的命令列表（都选「有响应映射」的只读命令，避免误判超时）
const TESTS = [
  {
    key: "role",
    cmd: "role_getroleinfo",
    label: "角色信息",
    make: () => ({ clientVersion: chosenVersion.value }),
  },
  { key: "activity", cmd: "activity_get", label: "活动", make: () => ({}) },
  {
    key: "databundle",
    cmd: "system_getdatabundlever",
    label: "数据包版本",
    make: () => ({ isAudit: false }),
  },
  { key: "tower", cmd: "tower_getinfo", label: "爬塔信息", make: () => ({}) },
];

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
  const wsUrl = token.wsUrl;
  if (!wsUrl) {
    message.error("该角色没有 wsUrl，请重新导入 token（扫码时勾选保存 wsUrl）");
    running.value = false;
    return;
  }

  pushStep(
    "info",
    "开始连接",
    `正在直接建立 WebSocket 连接…（clientVersion: ${chosenVersion.value}）`,
  );

  // 直接建干净连接，不走 store 的自动初始化，避免干扰诊断
  const client = new XyzwWebSocketClient({
    url: wsUrl,
    utils: g_utils,
    heartbeatMs: 60000,
  });
  myClient.value = client;
  client.setMessageListener(() => {}); // 不干扰内部响应处理
  client.init();

  const socket = client.socket;
  if (!socket) {
    pushStep("error", "连接未建立", "没有拿到 WebSocket 实例，请重试");
    running.value = false;
    return;
  }

  const onOpen = async () => {
    pushStep("success", "已连接", "WebSocket 握手成功（token 有效）");
    await runTests(client);
  };
  const onMessage = (evt) => {
    if (!firstMsg.value) {
      const pkt = readFrame(evt.data);
      const cmd = pkt?.cmd || pkt?.c || "?";
      const code = pkt?.code;
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
    const reasonText = evt.reason ? `（原因: ${evt.reason}）` : "（无原因文本）";
    const type = evt.code === 1000 ? "warning" : "error";
    pushStep(type, `连接被关闭 code=${evt.code}`, reasonText);
    finish();
  };

  if (socket.readyState === WebSocket.OPEN) onOpen();
  socket.addEventListener("open", onOpen);
  socket.addEventListener("message", onMessage);
  socket.addEventListener("close", onClose);

  // 兜底：8 秒还没结论，给个超时提示
  setTimeout(() => {
    if (
      running.value &&
      Object.keys(results.value).length === 0 &&
      !closeInfo.value
    ) {
      pushStep(
        "warning",
        "超时未关闭",
        "连接保持中，但 8 秒内未收到任何命令响应也未断开。可手动断开看结果。",
      );
      running.value = false;
    }
  }, 8000);
};

const runTests = async (client) => {
  for (const t of TESTS) {
    pushStep("info", `发送 ${t.label}`, `命令: ${t.cmd}`);
    try {
      const res = await client.sendWithPromise(t.cmd, t.make(), 6000);
      const name =
        res?.name || res?.roleName || (res?.role && res.role.name) || null;
      results.value[t.key] = {
        status: "ok",
        code: 0,
        desc: name ? `角色: ${name}` : "成功",
      };
      pushStep(
        "success",
        `${t.label}成功`,
        results.value[t.key].desc,
      );
    } catch (e) {
      const m = e?.message || String(e);
      const mm = m.match(/(\d{4,})/);
      const code = mm ? mm[1] : null;
      results.value[t.key] = { status: "error", code, desc: m };
      pushStep("error", `${t.label}失败`, m);
    }
  }
  finish();
};

const finish = () => {
  // 等 4 条命令都拿到结果，或连接已关闭，再给结论
  if (
    Object.keys(results.value).length < TESTS.length &&
    !closeInfo.value
  )
    return;
  if (running.value) running.value = false;

  const r = results.value;
  const roleOk = r.role?.status === "ok";
  const sessionOk =
    r.role?.status === "ok" ||
    r.activity?.status === "ok" ||
    r.databundle?.status === "ok";
  const allFail =
    Object.keys(r).length > 0 &&
    Object.values(r).every((x) => x.status === "error");

  // 连接建立后服务器直接关，没给任何命令机会
  if (Object.keys(r).length === 0 && closeInfo.value) {
    verdictTitle.value = "服务器直接踢连接";
    verdictStatus.value = "error";
    verdict.value =
      `连接建立后服务器以 ${closeInfo.value.code} 直接关闭，没给任何命令机会。\n` +
      `这是「整条会话被拒」的实锤。最可能：该账号在手机游戏/其他设备在线（游戏只允许一条会话），或游戏方对第三方 agent 接口做了连接层封禁。\n` +
      `👉 先彻底关闭手机上的咸鱼之王，再点「一键诊断」重测。`;
    return;
  }

  // 角色信息成功 → 连接完全正常
  if (roleOk) {
    verdictTitle.value = "连接正常 ✓";
    verdictStatus.value = "success";
    verdict.value =
      `角色信息（clientVersion ${chosenVersion.value}）成功返回，说明 token 与连接都正常。\n` +
      `你之前遇到的 200020 / 已断开，多半是：① 批跑时多个号同时连同一账号被互踢；② 某些活动/功能不在开放时间；③ 凌晨游戏结算窗口。\n` +
      `👉 白天、手机游戏关闭、单号单连的情况下，回去跑正式批任务即可。`;
    return;
  }

  // 部分命令成功 → 会话通，但角色类命令被选择性拒绝
  if (sessionOk) {
    const okList = Object.entries(r)
      .filter(([, v]) => v.status === "ok")
      .map(([k]) => k)
      .join(" / ");
    verdictTitle.value = "选择性拒绝（会话是通的）";
    verdictStatus.value = "warning";
    verdict.value =
      `部分命令成功（${okList}），但 role_getroleinfo / tower 被拒（${r.role?.code || r.tower?.code}）。\n` +
      `说明会话本身通，只是「角色信息 / 爬塔」这类命令被服务器拒——可能是需要前置握手（进入游戏/选服）或参数变了。\n` +
      `把结论发我，我查连接初始化是否缺步骤。`;
    return;
  }

  // 全部失败 → 整条会话被拒
  if (allFail) {
    verdictTitle.value = "整条会话被拒（最可能：同号互踢）";
    verdictStatus.value = "error";
    verdict.value =
      `角色信息 / 活动 / 数据包版本 / 爬塔 全部返回 ${r.role?.code || "错误码"}，说明游戏服务器在拒绝这条会话，不是单个命令的问题。\n` +
      `最可能：① 该账号在手机游戏/其他设备在线（游戏只允许一条会话），网页连接被挤掉；② 游戏方对第三方 agent 接口做了限制。\n` +
      `👉 先彻底关闭手机上的咸鱼之王，只留一个 ctt.ccwu.cc 标签页，再点「一键诊断」重测。`;
    return;
  }

  verdictTitle.value = "结果不完整";
  verdictStatus.value = "info";
  verdict.value = "结果不完整，把结论发我。";
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
  if (myClient.value?.socket) myClient.value.disconnect();
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
