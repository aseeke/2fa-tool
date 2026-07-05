<template>
  <main class="page-shell">
    <div class="ambient ambient-one"></div>
    <div class="ambient ambient-two"></div>
    <div class="ambient ambient-three"></div>

    <header class="topbar">
      <div class="hero-copy">
        <p class="eyebrow">Batch / TOTP</p>
        <h1>批量验证码生成器</h1>
        <p class="lede">每行一个 secret 或 otpauth:// URL，右侧自动分卡显示。</p>
      </div>
    </header>

    <section class="workspace">
      <article class="panel editor-panel">
        <div class="panel-head">
          <div>
            <p class="panel-kicker">输入</p>
            <h2>密钥或 otpauth 链接</h2>
          </div>
          <span class="counter">{{ entryCount }} 条</span>
        </div>

        <textarea
          v-model="input"
          class="input"
          rows="10"
          placeholder="每行一个 secret 或 otpauth:// URL"
          @focus="$event.target.select()"
        ></textarea>

        <div class="sample-row">
          <button
            v-for="sample in samplePresets"
            :key="sample.label"
            type="button"
            class="chip"
            @click="loadSample(sample.value)"
          >
            {{ sample.label }}
          </button>
        </div>

        <div class="action-row">
          <button type="button" class="primary-btn" @click="loadSample(demoBatch)">
            示例
          </button>
          <button type="button" class="secondary-btn" :disabled="!input.trim()" @click="clearInput">
            清空
          </button>
        </div>

        <div class="mini-strip">
          <span>{{ validCount }} 有效</span>
          <span>{{ invalidCount ? `${invalidCount} 错误` : '自动更新' }}</span>
          <span>本地生成</span>
        </div>
      </article>

      <article class="panel result-panel">
        <div class="result-head">
          <div>
            <p class="panel-kicker">结果</p>
            <h2>右侧卡片</h2>
          </div>
          <span class="counter">{{ cards.length }} 张</span>
        </div>

        <div v-if="cards.length" class="result-grid">
          <article
            v-for="card in cards"
            :key="card.id"
            class="token-card"
            :class="{ error: card.error, copied: copiedId === card.id }"
          >
            <div class="token-card-top">
              <span class="card-pill" :class="{ error: card.error }">
                {{ card.badge }}
              </span>
              <button
                type="button"
                class="icon-btn"
                :disabled="!card.code || Boolean(card.error)"
                @click="copyCode(card)"
              >
                {{ copiedId === card.id ? '已复制' : '复制' }}
              </button>
            </div>

            <p class="token-title">{{ card.title }}</p>

            <button
              type="button"
              class="token-code"
              :class="{ empty: !card.code || Boolean(card.error) }"
              :disabled="!card.code || Boolean(card.error)"
              @click="copyCode(card)"
              :aria-label="card.error ? '卡片错误' : '复制验证码'"
            >
              <span v-if="card.error" class="token-error">{{ card.error }}</span>
              <span v-else class="code-value">{{ card.code }}</span>
            </button>

            <div v-if="!card.error" class="token-meta">
              <span>{{ card.remain }} 秒</span>
              <span>{{ card.algorithm.toUpperCase() }}</span>
              <span>{{ card.digits }} 位</span>
              <span>{{ card.step }} 秒周期</span>
            </div>

            <p v-else class="token-note">检查这一行的格式</p>

            <div v-if="!card.error" class="progress-track" aria-hidden="true">
              <div
                class="progress-fill"
                :class="{ urgent: card.remain <= 5 }"
                :style="{ width: `${card.progress}%` }"
              ></div>
            </div>
          </article>
        </div>

        <div v-else class="empty-state">把内容贴到左边，右侧会自动展开成多张卡片。</div>
      </article>
    </section>
  </main>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';
import { createGuardrails, generateSync } from 'otplib';

const totpGuardrails = createGuardrails({ MIN_SECRET_BYTES: 10 });

const demoBatch = [
  'otpauth://totp/Demo-Account:root-account?secret=KZQW4ZLON5XW6ZLNMU3D4YJY&issuer=Demo',
  'otpauth://totp/Demo-Account:root-account?secret=KZQW4ZLON5XW6ZLNMU3D4YJY&issuer=Demo',
  'otpauth://totp/Demo-Account:root-account?secret=KZQW4ZLON5XW6ZLNMU3D4YJY&issuer=Demo',
  'otpauth://totp/Demo-Account:root-account?secret=KZQW4ZLON5XW6ZLNMU3D4YJY&issuer=Demo',
].join('\n');

const samplePresets = [
  {
    label: '4条',
    value: demoBatch,
  },
  {
    label: 'AWS',
    value: 'otpauth://totp/AWS:root-account?secret=JBSWY3DPEHPK3PXP&issuer=AWS',
  },
  {
    label: 'Base32',
    value: 'JBSWY3DPEHPK3PXP',
  },
];

const input = ref(demoBatch);
const copiedId = ref('');
const currentTime = ref(Date.now());

let clockId = null;
let copiedTimerId = null;

const beijingTimeFormatter = new Intl.DateTimeFormat('zh-CN', {
  timeZone: 'Asia/Shanghai',
  hour: '2-digit',
  minute: '2-digit',
  second: '2-digit',
  hour12: false,
});

const beijingDateFormatter = new Intl.DateTimeFormat('zh-CN', {
  timeZone: 'Asia/Shanghai',
  weekday: 'short',
  month: 'long',
  day: 'numeric',
});

function parseDigits(value) {
  if (!value) {
    return 6;
  }

  const digits = Number.parseInt(value, 10);
  if (![6, 7, 8].includes(digits)) {
    throw new Error('digits 仅支持 6、7、8');
  }

  return digits;
}

function parseStep(value) {
  if (!value) {
    return 30;
  }

  const step = Number.parseInt(value, 10);
  if (![15, 30, 60].includes(step)) {
    throw new Error('period 仅支持 15、30、60 秒');
  }

  return step;
}

function parseAlgorithm(value) {
  const algorithm = (value || 'sha1').toLowerCase();
  const validAlgos = ['sha1', 'sha256', 'sha512'];

  if (!validAlgos.includes(algorithm)) {
    throw new Error(`algorithm 仅支持 ${validAlgos.join(', ')}`);
  }

  return algorithm;
}

function createBaseEntry(index, rawLine) {
  return {
    id: `line-${index}-${rawLine}`,
    lineNumber: index + 1,
    raw: rawLine,
    source: 'secret',
    badge: 'Secret',
    title: `密钥 ${index + 1}`,
    subtitle: '手动输入',
    secret: '',
    digits: 6,
    step: 30,
    algorithm: 'sha1',
    error: '',
  };
}

function parseSecretLine(rawLine, index) {
  const entry = createBaseEntry(index, rawLine);
  entry.secret = rawLine.replace(/\s+/g, '').toUpperCase();

  if (!entry.secret) {
    entry.error = '密钥不能为空';
  }

  return entry;
}

function parseOtpauthLine(rawLine, index) {
  const entry = createBaseEntry(index, rawLine);

  try {
    const url = new URL(rawLine);

    if (url.hostname.toLowerCase() !== 'totp') {
      throw new Error('仅支持 otpauth://totp/');
    }

    const secret = url.searchParams.get('secret');
    if (!secret) {
      throw new Error('缺少 secret 参数');
    }

    const digits = parseDigits(url.searchParams.get('digits'));
    const step = parseStep(url.searchParams.get('period'));
    const algorithm = parseAlgorithm(url.searchParams.get('algorithm'));
    const issuer = url.searchParams.get('issuer')?.trim() || '';
    const decodedPath = decodeURIComponent(url.pathname.replace(/^\/+/, '')).trim();
    const pathParts = decodedPath.split(':').map((part) => part.trim()).filter(Boolean);
    const pathIssuer = pathParts.length > 1 ? pathParts[0] : '';
    const account = pathParts.length > 1 ? pathParts.slice(1).join(':') : decodedPath;

    entry.source = 'otpauth';
    entry.badge = issuer || pathIssuer || 'otpauth';
    entry.title = account || issuer || `账户 ${index + 1}`;
    entry.subtitle = 'otpauth:// URL';
    entry.secret = secret.replace(/\s+/g, '').toUpperCase();
    entry.digits = digits;
    entry.step = step;
    entry.algorithm = algorithm;
  } catch (err) {
    entry.error = `解析失败：${err.message}`;
  }

  return entry;
}

function parseBatch(value) {
  return value
    .split(/\r?\n/)
    .map((line, index) => line.trim())
    .filter((line) => line && !line.startsWith('#'))
    .map((line, index) => {
      if (line.startsWith('otpauth://')) {
        return parseOtpauthLine(line, index);
      }

      return parseSecretLine(line, index);
    });
}

function generateCode(entry) {
  return generateSync({
    secret: entry.secret,
    digits: entry.digits,
    period: entry.step,
    algorithm: entry.algorithm,
    guardrails: totpGuardrails,
  });
}

function buildCard(entry, now) {
  if (entry.error) {
    return {
      ...entry,
      code: '',
      remain: 0,
      progress: 0,
    };
  }

  const epoch = now / 1000;
  const elapsed = epoch % entry.step;

  let code = '';
  try {
    code = generateCode(entry);
  } catch (err) {
    return {
      ...entry,
      error: err.message,
      code: '',
      remain: 0,
      progress: 0,
    };
  }

  return {
    ...entry,
    code,
    remain: Math.max(1, Math.ceil(entry.step - elapsed)),
    progress: (elapsed / entry.step) * 100,
  };
}

const parsedEntries = computed(() => parseBatch(input.value));
const cards = computed(() => parsedEntries.value.map((entry) => buildCard(entry, currentTime.value)));
const entryCount = computed(() => parsedEntries.value.length);
const validCount = computed(() => cards.value.filter((card) => !card.error).length);
const invalidCount = computed(() => cards.value.filter((card) => card.error).length);
const beijingTime = computed(() => beijingTimeFormatter.format(new Date(currentTime.value)));
const beijingDate = computed(() => beijingDateFormatter.format(new Date(currentTime.value)));

function loadSample(value) {
  input.value = value;
}

function clearInput() {
  input.value = '';
}

async function copyCode(card) {
  if (!card.code) {
    return;
  }

  try {
    await navigator.clipboard.writeText(card.code);
    copiedId.value = card.id;
    clearTimeout(copiedTimerId);
    copiedTimerId = setTimeout(() => {
      if (copiedId.value === card.id) {
        copiedId.value = '';
      }
    }, 1200);
  } catch {
    copiedId.value = '';
  }
}

watch(input, () => {
  copiedId.value = '';
  clearTimeout(copiedTimerId);
});

onMounted(() => {
  clockId = setInterval(() => {
    currentTime.value = Date.now();
  }, 250);
});

onUnmounted(() => {
  clearInterval(clockId);
  clearTimeout(copiedTimerId);
});
</script>
