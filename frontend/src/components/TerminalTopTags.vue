<script setup lang="ts">
import { t } from "@/lang/i18n";
import {
  ApartmentOutlined,
  BlockOutlined,
  DashboardOutlined,
  HddOutlined
} from "@ant-design/icons-vue";
import { useLocalStorage } from "@vueuse/core";
import prettyBytes, { type Options as PrettyOptions } from "pretty-bytes";
import { computed, type Component } from "vue";

interface TerminalRuntimeInfo {
  cpuUsage?: number;
  cpuLimit?: number;
  memoryUsage?: number;
  memoryLimit?: number;
  memoryUsagePercent?: number;
  rxBytes?: number;
  txBytes?: number;
  rxRate?: number;
  txRate?: number;
  networkInterfaces?: string[];
  networkStatsSource?: "docker";
  storageUsage?: number;
  storageLimit?: number;
}

interface TerminalRuntimeConfig {
  processType?: string;
  docker?: {
    cpuLimit?: number;
    cpuUsage?: number;
    memory?: number;
    maxSpace?: number;
    image?: string;
  };
}

interface PerfCardItem {
  key: string;
  label: string;
  value: string;
  icon: Component;
  theme: string;
  barPercent: number;
  onClick?: () => void;
}

const props = defineProps<{
  info?: TerminalRuntimeInfo | null;
  config?: TerminalRuntimeConfig | null;
  isStopped: boolean;
}>();

const useByteUnit = useLocalStorage("useByteUnit", true);
const prettyBytesConfig: PrettyOptions = {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
  binary: true
};

const unlimitedText = () => t("TXT_CODE_a92df201");

const toSafeNumber = (value?: number) =>
  typeof value === "number" && Number.isFinite(value) ? Math.max(value, 0) : 0;

const formatPercentUsage = (usage?: number, limit?: number) => {
  const fUsage = `${Math.round(toSafeNumber(usage))}%`;
  const fLimit = typeof limit === "number" && limit > 0 ? `${Math.round(limit)}%` : unlimitedText();
  return `${fUsage} / ${fLimit}`;
};

const formatByteUsage = (usage?: number, limit?: number) => {
  const fUsage = prettyBytes(toSafeNumber(usage), prettyBytesConfig);
  const fLimit =
    typeof limit === "number" && limit > 0
      ? prettyBytes(limit, prettyBytesConfig)
      : unlimitedText();
  return `${fUsage} / ${fLimit}`;
};

const getUsagePercent = (usage?: number, limit?: number) => {
  const safeUsage = toSafeNumber(usage);
  if (typeof limit === "number" && limit > 0) {
    return Math.min((safeUsage / limit) * 100, 100);
  }
  return Math.min(safeUsage, 100);
};

const getCpuStatus = (usage?: number, limit?: number) => {
  if (typeof limit === "number" && limit > 0) {
    const percent = getUsagePercent(usage, limit);
    if (percent >= 90) return "error";
    if (percent >= 75) return "warning";
    return "normal";
  }
  return usage != null && usage > 600
    ? "error"
    : usage != null && usage > 200
    ? "warning"
    : "normal";
};

const isDockerInstance = (config?: TerminalRuntimeConfig | null) =>
  config?.processType === "docker" || Boolean(config?.docker?.image);

const getConfigCpuLimit = (config?: TerminalRuntimeConfig | null) => {
  const docker = config?.docker;
  if (typeof docker?.cpuLimit === "number" && docker.cpuLimit > 0) {
    return Math.round(docker.cpuLimit * 100);
  }
  if (typeof docker?.cpuUsage === "number" && docker.cpuUsage > 0) {
    return docker.cpuUsage;
  }
  return undefined;
};

const getConfigMemoryLimit = (config?: TerminalRuntimeConfig | null) => {
  const memory = config?.docker?.memory;
  return typeof memory === "number" && memory > 0 ? memory * 1024 * 1024 : undefined;
};

const getConfigStorageLimit = (config?: TerminalRuntimeConfig | null) => {
  const maxSpace = config?.docker?.maxSpace;
  return typeof maxSpace === "number" && maxSpace > 0 ? maxSpace * 1024 * 1024 * 1024 : undefined;
};

const formatNetworkSpeed = (bytes?: number) =>
  useByteUnit.value
    ? prettyBytes(bytes ?? 0, { ...prettyBytesConfig, binary: false }) + "/s"
    : prettyBytes((bytes ?? 0) * 8, { ...prettyBytesConfig, bits: true, binary: false }).replace(
        /bit$/,
        "b"
      ) + "ps";

const formatTrafficUsage = (bytes?: number) =>
  prettyBytes(bytes ?? 0, { ...prettyBytesConfig, binary: false });

const cards = computed<PerfCardItem[]>(() => {
  const info = props.info ?? {};
  const showConfiguredResources = isDockerInstance(props.config);
  if (!props.info && !showConfiguredResources) return [];

  const {
    cpuUsage,
    cpuLimit,
    memoryUsage,
    memoryLimit,
    memoryUsagePercent,
    rxBytes,
    txBytes,
    rxRate,
    txRate,
    storageUsage,
    storageLimit
  } = info;

  const finalCpuUsage = cpuUsage ?? (showConfiguredResources ? 0 : undefined);
  const finalCpuLimit = cpuLimit ?? getConfigCpuLimit(props.config);
  const finalMemoryUsage = memoryUsage ?? (showConfiguredResources ? 0 : undefined);
  const finalMemoryLimit = getConfigMemoryLimit(props.config) ?? memoryLimit;
  const finalStorageUsage = storageUsage ?? (showConfiguredResources ? 0 : undefined);
  const finalStorageLimit = storageLimit ?? getConfigStorageLimit(props.config);

  const cpuStatus = getCpuStatus(finalCpuUsage, finalCpuLimit);
  const cpuPercent = getUsagePercent(finalCpuUsage, finalCpuLimit);
  const memPercent =
    finalMemoryUsage != null && finalMemoryLimit
      ? getUsagePercent(finalMemoryUsage, finalMemoryLimit)
      : Math.min(memoryUsagePercent ?? 0, 100);
  const storagePercent = getUsagePercent(finalStorageUsage, finalStorageLimit);

  const items: (PerfCardItem | null)[] = [
    finalCpuUsage != null
      ? {
          key: "cpu",
          label: t("TXT_CODE_b862a158"),
          value: formatPercentUsage(finalCpuUsage, finalCpuLimit),
          icon: BlockOutlined,
          theme: `perf-card--cpu-${cpuStatus}`,
          barPercent: cpuPercent
        }
      : null,

    finalMemoryUsage != null
      ? {
          key: "memory",
          label: t("TXT_CODE_593ee330"),
          value: formatByteUsage(finalMemoryUsage, finalMemoryLimit),
          icon: DashboardOutlined,
          theme: "perf-card--memory",
          barPercent: memPercent
        }
      : null,

    finalStorageUsage != null
      ? {
          key: "disk",
          label: t("TXT_CODE_DISK_USAGE"),
          value: formatByteUsage(finalStorageUsage, finalStorageLimit),
          icon: HddOutlined,
          theme: "perf-card--disk",
          barPercent: storagePercent
        }
      : null,

    rxRate != null || txRate != null
      ? {
          key: "network-bandwidth",
          label: `${t("TXT_CODE_50daec4")} · 带宽`,
          value: `↓${formatNetworkSpeed(rxRate)} ↑${formatNetworkSpeed(txRate)}`,
          icon: ApartmentOutlined,
          theme: "perf-card--network",
          barPercent: 0,
          onClick: () => {
            useByteUnit.value = !useByteUnit.value;
          }
        }
      : null,

    rxBytes != null || txBytes != null
      ? {
          key: "network-traffic",
          label: `${t("TXT_CODE_50daec4")} · 流量`,
          value: `↓${formatTrafficUsage(rxBytes)} ↑${formatTrafficUsage(txBytes)}`,
          icon: ApartmentOutlined,
          theme: "perf-card--network",
          barPercent: 0
        }
      : null
  ];

  return items.filter(Boolean) as PerfCardItem[];
});
</script>

<template>
  <div v-if="cards.length > 0" class="perf-cards">
    <div
      v-for="card in cards"
      :key="card.key"
      class="perf-card"
      :class="card.theme"
      @click="card.onClick?.()"
    >
      <div class="perf-card__body">
        <div class="perf-card__icon">
          <component :is="card.icon" />
        </div>
        <div class="perf-card__info">
          <span class="perf-card__label">{{ card.label }}</span>
          <span class="perf-card__value">{{ card.value }}</span>
        </div>
      </div>
      <div class="perf-card__bar-track">
        <div class="perf-card__bar" :style="{ width: card.barPercent + '%' }"></div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.perf-cards {
  display: flex;
  flex-wrap: wrap;
  justify-content: flex-start;
  align-items: center;
  gap: 6px;
}

.perf-card {
  user-select: none;
  min-width: 170px;
  height: 48px;
  border-radius: 8px;
  opacity: 0.9;
  background: var(--color-gray-1);
  border: 1px solid var(--color-gray-4);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  transition:
    box-shadow 0.2s ease,
    transform 0.2s ease;

  &:hover {
    box-shadow: 0 4px 12px var(--card-shadow-color);
    transform: translateY(-1px);
  }
}

.perf-card__body {
  flex: 1;
  display: flex;
  align-items: center;
  padding: 5px 10px 0 8px;
  gap: 8px;
  min-width: 0;
}

.perf-card__icon {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  flex-shrink: 0;
}

.perf-card__info {
  display: flex;
  flex-direction: column;
  min-width: 0;
  gap: 2px;
  overflow: hidden;
}

.perf-card__label {
  font-size: 10px;
  line-height: 1;
  color: var(--color-gray-7);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.perf-card__value {
  font-size: 11px;
  font-weight: 600;
  line-height: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.perf-card__bar-track {
  height: 3px;
  background: var(--color-gray-4);
  overflow: hidden;
}

.perf-card__bar {
  height: 100%;
  border-radius: 2px;
  transition: width 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ---- CPU normal (blue) ---- */
.perf-card--cpu-normal {
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-blue-1);
    color: var(--color-blue-5);
  }
  .perf-card__value {
    color: var(--color-blue-7);
  }
  .perf-card__bar {
    background: var(--color-blue-5);
  }
}

/* ---- CPU warning (gold) ---- */
.perf-card--cpu-warning {
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-gold-1);
    color: var(--color-gold-6);
  }
  .perf-card__value {
    color: var(--color-gold-7);
  }
  .perf-card__bar {
    background: var(--color-gold-5);
    animation: bar-pulse 1.5s ease-in-out infinite;
  }
}

/* ---- CPU error (red) ---- */
.perf-card--cpu-error {
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-red-1);
    color: var(--color-red-5);
    animation: icon-pulse 1s ease-in-out infinite;
  }
  .perf-card__value {
    color: var(--color-red-6);
  }
  .perf-card__bar {
    background: var(--color-red-5);
    animation: bar-pulse 0.8s ease-in-out infinite;
  }
}

/* ---- Memory (purple) ---- */
.perf-card--memory {
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-purple-1);
    color: var(--color-purple-5);
  }
  .perf-card__value {
    color: var(--color-purple-6);
  }
  .perf-card__bar {
    background: var(--color-purple-5);
  }
}

/* ---- Disk (cyan) ---- */
.perf-card--disk {
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-cyan-1);
    color: var(--color-cyan-6);
  }
  .perf-card__value {
    color: var(--color-cyan-7);
  }
  .perf-card__bar {
    background: var(--color-cyan-5);
  }
}

/* ---- Network (green) ---- */
.perf-card--network {
  cursor: pointer;
  border-color: var(--color-gray-5);
  .perf-card__icon {
    background: var(--color-green-1);
    color: var(--color-green-6);
  }
  .perf-card__value {
    color: var(--color-green-7);
  }
}

.perf-card__bar--flow {
  width: 35%;
  background: linear-gradient(
    90deg,
    transparent 0%,
    var(--color-green-4) 40%,
    var(--color-green-5) 60%,
    transparent 100%
  );
  animation: bar-flow 2s ease-in-out infinite;
}

@keyframes bar-flow {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(400%);
  }
}

@keyframes bar-pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.55;
  }
}

@keyframes icon-pulse {
  0%,
  100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.15);
    opacity: 0.7;
  }
}
</style>
