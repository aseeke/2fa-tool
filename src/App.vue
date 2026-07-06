<template>
  <main class="page-shell">
    <div class="ambient ambient-a" aria-hidden="true"></div>
    <div class="ambient ambient-b" aria-hidden="true"></div>
    <div class="ambient ambient-c" aria-hidden="true"></div>

    <header class="hero">
      <div class="hero-copy">
        <p class="eyebrow">
          <span class="brand-mark" aria-hidden="true"></span>
          Browser-only TOTP
        </p>
        <h1>2FA / MFA 验证码</h1>
        <p class="lede">
          兼容 Oracle、AWS、Google Authenticator 常见 TOTP 验证码。密钥只在本次请求中计算，不写入数据库。
        </p>

        <div class="hero-tags" aria-label="功能标签">
          <span>本地计算</span>
          <span>支持 otpauth://</span>
          <span>支持 Base32 secret</span>
          <span>多行批量输入</span>
        </div>
      </div>

      <aside class="clock-card" aria-label="北京时间">
        <span class="clock-label">北京时间</span>
        <strong class="clock-time">{{ beijingTime }}</strong>
        <span class="clock-date">{{ beijingDate }}</span>
      </aside>
    </header>

    <section class="workspace">
      <article class="panel input-panel">
        <div class="panel-head">
          <div>
            <p class="panel-kicker">输入</p>
            <h2>密钥或 otpauth 链接</h2>
          </div>
          <span class="counter">{{ draftCount }} 行</span>
        </div>

        <textarea
          v-model="draftInput"
          class="input"
          rows="10"
          placeholder="每行一个，例如：&#10;JBSWY3DPEHPK3PXP&#10;otpauth://totp/AWS:root-account?secret=JBSWY3DPEHPK3PXP&issuer=AWS"
          spellcheck="false"
        ></textarea>

        <div class="action-row">
          <button type="button" class="primary-btn" :disabled="!draftInput.trim()" @click="generateTokens">
            生成验证码
          </button>
          <button
            type="button"
            class="close-btn"
            :disabled="!draftInput && !submittedInput"
            @click="clearAll"
            aria-label="清空"
            title="清空"
          >
            ×
          </button>
        </div>

        <div class="mini-strip">
          <span>{{ validCount }} 可用</span>
          <span>{{ invalidCount ? `${invalidCount} 无效` : '等待生成' }}</span>
          <span>仅浏览器本地计算</span>
        </div>

        <details class="api-details">
          <summary>
            <span>API</span>
            <small>默认折叠</small>
          </summary>

          <div class="api-body">
            <p class="api-note">
              这个页面本身不写数据库，也不依赖后端。下面是你如果要接服务端时，可以直接复用的参数格式。
            </p>

            <div v-for="example in apiExamples" :key="example.title" class="api-block">
              <div class="api-block-head">
                <strong>{{ example.title }}</strong>
                <button type="button" class="copy-btn" @click="copyText(example.sample)">
                  复制
                </button>
              </div>
              <pre>{{ example.sample }}</pre>
              <p>{{ example.description }}</p>
            </div>
          </div>
        </details>
      </article>

      <article class="panel result-panel">
        <div class="panel-head">
          <div>
            <p class="panel-kicker">结果</p>
            <h2>验证码展示区</h2>
          </div>
          <span class="counter">{{ cards.length }} 项</span>
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
            <p v-if="card.subtitle" class="token-subtitle">{{ card.subtitle }}</p>

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

            <p v-else class="token-note">请检查这一行的格式或 secret 内容。</p>

            <div v-if="!card.error" class="progress-track" aria-hidden="true">
              <div
                class="progress-fill"
                :class="{ urgent: card.remain <= 5 }"
                :style="{ width: `${card.progress}%` }"
              ></div>
            </div>
          </article>
        </div>

        <div v-else class="empty-state">
          输入密钥后点击「生成验证码」，右侧会显示每一条 TOTP。
        </div>
      </article>
    </section>
  </main>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';
import { generateSync } from 'otplib';

const draftInput = ref('');
const submittedInput = ref('');
const copiedId = ref('');
const now = ref(Date.now());

let clockId = null;
let copiedTimerId = null;

const apiExamples = [
  {
    title: 'GET /api/totp',
    sample: 'GET /api/totp?secret=BASE32&digits=6&period=30&algorithm=sha1',
    description: '单个密钥生成当前验证码，适合做成轻量查询接口。',
  },
  {
    title: 'POST /api/totp/batch',
    sample: 'POST /api/totp/batch\n{ "items": ["JBSWY3DPEHPK3PXP", "otpauth://totp/AWS:root?secret=..."] }',
    description: '批量输入，返回每条记录对应的解析结果和当前验证码。',
  },
  {
    title: 'GET /api/time',
    sample: 'GET /api/time',
    description: '返回北京时间，和页面右上角保持一致。',
  },
];

function formatBeijingTime(value) {
  return new Intl.DateTimeFormat('zh-CN', {
    timeZone: 'Asia/Shanghai',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  }).format(value);
}

function formatBeijingDate(value) {
  return new Intl.DateTimeFormat('zh-CN', {
    timeZone: 'Asia/Shanghai',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    weekday: 'short',
  }).format(value);
}

function sanitizeSecret(value) {
  return value.replace(/[\s-]/g, '').toUpperCase();
}

function parseDigits(value) {
  if (!value) return 6;

  const digits = Number.parseInt(value, 10);
  if (![6, 7, 8].includes(digits)) {
    throw new Error('digits 仅支持 6、7、8');
  }

  return digits;
}

function parseStep(value) {
  if (!value) return 30;

  const step = Number.parseInt(value, 10);
  if (!Number.isFinite(step) || step < 15 || step > 120) {
    throw new Error('period 需要在 15 到 120 秒之间');
  }

  return step;
}

function parseAlgorithm(value) {
  const algorithm = (value || 'sha1').toLowerCase();
  const validAlgorithms = ['sha1', 'sha256', 'sha512'];

  if (!validAlgorithms.includes(algorithm)) {
    throw new Error(`algorithm 仅支持 ${validAlgorithms.join(', ')}`);
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
  entry.secret = sanitizeSecret(rawLine);

  if (!entry.secret) {
    entry.error = '密钥不能为空';
  }

  return entry;
}

function parseOtpauthLine(rawLine, index) {
  const entry = createBaseEntry(index, rawLine);

  try {
    const url = new URL(rawLine);

    if (url.protocol !== 'otpauth:' || url.hostname.toLowerCase() !== 'totp') {
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
    entry.secret = sanitizeSecret(secret);
    entry.digits = digits;
    entry.step = step;
    entry.algorithm = algorithm;
  } catch (error) {
    entry.error = `解析失败：${error.message}`;
  }

  return entry;
}

function parseBatch(value) {
  return value
    .split(/\r?\n/)
    .map((line, index) => line.trim())
    .filter((line) => line && !line.startsWith('#'))
    .map((line, index) => (line.startsWith('otpauth://') ? parseOtpauthLine(line, index) : parseSecretLine(line, index)));
}

function generateCode(entry) {
  return generateSync({
    secret: entry.secret,
    digits: entry.digits,
    period: entry.step,
    algorithm: entry.algorithm,
  });
}

function buildCard(entry, currentTime) {
  if (entry.error) {
    return {
      ...entry,
      code: '',
      remain: 0,
      progress: 0,
    };
  }

  const epoch = Math.floor(currentTime / 1000);
  const elapsed = epoch % entry.step;

  try {
    const code = generateCode(entry);

    return {
      ...entry,
      code,
      remain: Math.max(1, entry.step - elapsed),
      progress: (elapsed / entry.step) * 100,
    };
  } catch (error) {
    return {
      ...entry,
      error: error.message,
      code: '',
      remain: 0,
      progress: 0,
    };
  }
}

const parsedEntries = computed(() => parseBatch(submittedInput.value));
const cards = computed(() => parsedEntries.value.map((entry) => buildCard(entry, now.value)));
const draftCount = computed(() => parseBatch(draftInput.value).length);
const validCount = computed(() => cards.value.filter((card) => !card.error).length);
const invalidCount = computed(() => cards.value.filter((card) => card.error).length);
const beijingTime = computed(() => formatBeijingTime(now.value));
const beijingDate = computed(() => formatBeijingDate(now.value));

function generateTokens() {
  submittedInput.value = draftInput.value.trim();
}

function clearAll() {
  draftInput.value = '';
  submittedInput.value = '';
  copiedId.value = '';
  clearTimeout(copiedTimerId);
}

async function copyText(text) {
  try {
    await navigator.clipboard.writeText(text);
  } catch {
    // Clipboard access can fail in some browser contexts.
  }
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

watch(draftInput, () => {
  copiedId.value = '';
  clearTimeout(copiedTimerId);
});

onMounted(() => {
  clockId = setInterval(() => {
    now.value = Date.now();
  }, 1000);
});

onUnmounted(() => {
  clearInterval(clockId);
  clearTimeout(copiedTimerId);
});
</script>
