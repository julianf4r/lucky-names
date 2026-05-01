<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch, reactive } from 'vue';
import LuckyWheel from './components/LuckyWheel.vue';
import RightPanel from './components/RightPanel.vue';

interface HistoryEntry {
    id: string;
    name: string;
    date: number;
}

const STORAGE_KEYS = {
    names: 'lucky-names-input',
    history: 'lucky-names-history',
    settings: 'lucky-names-settings',
} as const;

const DEFAULT_WHEEL_SETTINGS = {
    fontSize: 1.5,
    textRadius: 0.6,
    textWidth: 0.8,
} as const;

const SETTING_RANGES = {
    fontSize: { min: 0.5, max: 2.0 },
    textRadius: { min: 0.4, max: 0.8 },
    textWidth: { min: 0.4, max: 1.0 },
} as const;

type WheelSettingKey = keyof typeof DEFAULT_WHEEL_SETTINGS;

const namesInput = ref('林舟\n顾澈\n沈野\n陈若松\n苏临舟\n明砚');
const history = ref<HistoryEntry[]>([]);
const activeTab = ref<'names' | 'history'>('names');

const wheelSettings = reactive<Record<WheelSettingKey, number>>({ ...DEFAULT_WHEEL_SETTINGS });

const isSettingsPanelOpen = ref(false);
const selectedWinner = ref('');
const isWinnerDialogOpen = ref(false);
const rightPanelRef = ref<HTMLElement | null>(null);
const settingsBtnRef = ref<HTMLElement | null>(null);

const toggleSettingsPanel = () => {
    isSettingsPanelOpen.value = !isSettingsPanelOpen.value;
};

const handleClickOutside = (event: MouseEvent) => {
    if (isSettingsPanelOpen.value) {
        const target = event.target as Node;
        const isClickInsidePanel = rightPanelRef.value?.contains(target);
        const isClickOnButton = settingsBtnRef.value?.contains(target);
        if (!isClickInsidePanel && !isClickOnButton) {
            isSettingsPanelOpen.value = false;
        }
    }
};

watch(isSettingsPanelOpen, (isOpen) => {
    if (isOpen) {
        document.addEventListener('mousedown', handleClickOutside);
    } else {
        document.removeEventListener('mousedown', handleClickOutside);
    }
});

onUnmounted(() => {
    document.removeEventListener('mousedown', handleClickOutside);
});

const names = computed(() => {
    return namesInput.value.split('\n').filter(name => name.trim() !== '').map(name => name.trim());
});

const createHistoryId = () => {
    return `${Date.now()}-${crypto.randomUUID?.() ?? Math.random().toString(36).slice(2)}`;
};

const handleWinnerSelected = (winner: string) => {
    history.value.unshift({ id: createHistoryId(), name: winner, date: Date.now() });
    selectedWinner.value = winner;
    isWinnerDialogOpen.value = true;
};

const closeWinnerDialog = () => {
    isWinnerDialogOpen.value = false;
};

const clearHistory = () => {
    if (window.confirm('确定要清空所有历史记录吗？')) {
        history.value = [];
    }
};

const formatDate = (timestamp: number) => {
    const date = new Date(timestamp);
    const Y = date.getFullYear();
    const M = (date.getMonth() + 1).toString().padStart(2, '0');
    const D = date.getDate().toString().padStart(2, '0');
    const h = date.getHours().toString().padStart(2, '0');
    const m = date.getMinutes().toString().padStart(2, '0');
    const s = date.getSeconds().toString().padStart(2, '0');
    return `${Y}-${M}-${D} ${h}:${m}:${s}`;
};

const safeJsonParse = (value: string | null): unknown => {
    if (!value) return null;
    try {
        return JSON.parse(value);
    } catch {
        return null;
    }
};

const isFiniteNumber = (value: unknown): value is number => {
    return typeof value === 'number' && Number.isFinite(value);
};

const clamp = (value: number, min: number, max: number) => {
    return Math.min(max, Math.max(min, value));
};

const normalizeHistory = (value: unknown): HistoryEntry[] => {
    if (!Array.isArray(value)) return [];

    return value.flatMap((entry) => {
        if (typeof entry === 'string' && entry.trim() !== '') {
            return [{
                id: createHistoryId(),
                name: entry.trim(),
                date: Date.now(),
            }];
        }

        if (
            entry &&
            typeof entry === 'object' &&
            'name' in entry &&
            'date' in entry &&
            typeof entry.name === 'string' &&
            entry.name.trim() !== '' &&
            isFiniteNumber(entry.date)
        ) {
            return [{
                id: 'id' in entry && typeof entry.id === 'string' && entry.id !== ''
                    ? entry.id
                    : createHistoryId(),
                name: entry.name.trim(),
                date: entry.date,
            }];
        }

        return [];
    });
};

const applySavedSettings = (value: unknown) => {
    if (!value || typeof value !== 'object') return;

    const savedSettings = value as Partial<Record<WheelSettingKey, unknown>>;
    const settingKeys = Object.keys(DEFAULT_WHEEL_SETTINGS) as WheelSettingKey[];

    settingKeys.forEach((key) => {
        const savedValue = savedSettings[key];
        if (!isFiniteNumber(savedValue)) return;

        const range = SETTING_RANGES[key];
        wheelSettings[key] = clamp(savedValue, range.min, range.max);
    });
};

onMounted(() => {
    const savedNames = localStorage.getItem(STORAGE_KEYS.names);
    if (savedNames !== null) {
        namesInput.value = savedNames;
    }

    history.value = normalizeHistory(safeJsonParse(localStorage.getItem(STORAGE_KEYS.history)));
    applySavedSettings(safeJsonParse(localStorage.getItem(STORAGE_KEYS.settings)));
});

watch(namesInput, (newValue) => {
    localStorage.setItem(STORAGE_KEYS.names, newValue);
});
watch(history, (newValue) => {
    localStorage.setItem(STORAGE_KEYS.history, JSON.stringify(newValue));
}, { deep: true });
watch(wheelSettings, (newValue) => {
    localStorage.setItem(STORAGE_KEYS.settings, JSON.stringify(newValue));
});
</script>

<template>
    <div id="app">
        <div class="sidebar">
            <div class="tabs">
                <button @click="activeTab = 'names'" :class="{ active: activeTab === 'names' }">名单</button>
                <button @click="activeTab = 'history'" :class="{ active: activeTab === 'history' }">历史</button>
            </div>
            <div class="tab-content">
                <div v-if="activeTab === 'names'" class="names-panel">
                    <textarea v-model="namesInput" placeholder="请输入参与抽奖的名单，每行一个"></textarea>
                </div>
                <div v-if="activeTab === 'history'" class="history-panel">
                    <div class="history-header">
                        <h3>历史记录</h3>
                        <button @click="clearHistory" class="clear-btn" v-if="history.length > 0">清空记录</button>
                    </div>
                    <ul v-if="history.length > 0">
                        <li v-for="entry in history" :key="entry.id">
                            <span class="history-name">{{ entry.name }}</span>
                            <span class="history-date">{{ formatDate(entry.date) }}</span>
                        </li>
                    </ul>
                    <p v-else class="no-history">暂无历史记录</p>
                </div>
            </div>
        </div>
        <div class="main-content" :class="{ 'shifted-left': isSettingsPanelOpen }">
            <button @click="toggleSettingsPanel" class="settings-btn" ref="settingsBtnRef">
                <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"
                    fill="currentColor">
                    <path d="M0 0h24v24H0V0z" fill="none" />
                    <path
                        d="M12 8c1.1 0 2-.9 2-2s-.9-2-2-2-2 .9-2 2 .9 2 2 2zm0 2c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm0 6c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2z" />
                </svg>
            </button>
            <LuckyWheel :prizes="names" :font-size-multiplier="wheelSettings.fontSize"
                :text-radius-factor="wheelSettings.textRadius" :text-width-multiplier="wheelSettings.textWidth"
                @winner-selected="handleWinnerSelected" />
        </div>
        <Transition name="slide-fade">
            <div v-if="isSettingsPanelOpen" class="right-panel" ref="rightPanelRef">
                <RightPanel v-model:font-size="wheelSettings.fontSize" v-model:text-radius="wheelSettings.textRadius"
                    v-model:text-width="wheelSettings.textWidth" />
            </div>
        </Transition>
        <Transition name="winner-fade">
            <div v-if="isWinnerDialogOpen" class="winner-overlay" @click.self="closeWinnerDialog">
                <div class="fireworks">
                    <span v-for="index in 18" :key="index" class="spark" :style="{ '--i': index }"></span>
                </div>
                <div class="winner-dialog">
                    <p class="winner-kicker">恭喜中奖</p>
                    <h2>{{ selectedWinner }}</h2>
                    <button class="winner-close-btn" @click="closeWinnerDialog">知道了</button>
                </div>
            </div>
        </Transition>
    </div>
</template>

<style scoped>
#app {
    display: flex;
    height: 100vh;
    background-color: var(--light-gray);
    color: var(--text-color);
    position: relative;
    overflow: hidden;
}

.sidebar {
    width: 320px;
    background-color: #ffffff;
    border-right: 1px solid var(--border-color);
    display: flex;
    flex-direction: column;
    box-shadow: none;
    padding: 0 16px;
    flex-shrink: 0;
    z-index: 10;
    box-sizing: border-box;
}

.main-content {
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: var(--light-gray);
    padding: 40px;
    box-sizing: border-box;
    min-width: 0;
    position: relative;
    transition: padding-right 0.4s ease;
}

.main-content.shifted-left {
    padding-right: 320px;
}

.settings-btn {
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 20;
    background-color: #fff;
    border: 1px solid var(--border-color);
    border-radius: 50%;
    width: 44px;
    height: 44px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    color: var(--dark-gray);
    transition: all 0.2s;
}

.settings-btn:hover {
    color: var(--primary-color);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.right-panel {
    position: absolute;
    top: 0;
    right: 0;
    width: 280px;
    height: 100%;
    background-color: #ffffff;
    border-left: 1px solid var(--border-color);
    padding: 20px;
    box-shadow: -4px 0 20px rgba(0, 0, 0, 0.08);
    z-index: 100;
    box-sizing: border-box;
}

.slide-fade-enter-active,
.slide-fade-leave-active {
    transition: transform 0.3s ease;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
    transform: translateX(100%);
}

.tabs {
    display: flex;
    padding-top: 8px;
}

.tabs button {
    flex: 1;
    padding: 10px 14px;
    border: none;
    background-color: transparent;
    cursor: pointer;
    font-size: 16px;
    font-weight: 500;
    color: var(--dark-gray);
    border-radius: 8px;
    transition: background-color 0.2s, color 0.2s;
}

.tabs button.active {
    background-color: var(--primary-color);
    color: #fff;
    font-weight: 500;
}

.tab-content {
    flex: 1;
    padding: 16px 0 20px;
    overflow-y: auto;
    min-height: 0;
}

.names-panel {
    height: 100%;
}

.names-panel textarea {
    width: 100%;
    height: 100%;
    border: none;
    background-color: var(--light-gray);
    border-radius: 8px;
    padding: 12px;
    font-size: 15px;
    font-family: 'Inter', 'Microsoft YaHei', 'PingFang SC', 'Hiragino Sans GB', sans-serif;
    resize: none;
    box-sizing: border-box;
    transition: border-color 0.2s, box-shadow 0.2s;
}

.names-panel textarea:focus {
    outline: none;
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.2);
}

.history-panel {
    display: flex;
    flex-direction: column;
    height: 100%;
}

.history-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 4px;
}

.history-panel h3 {
    margin: 0;
    font-size: 18px;
    font-weight: 700;
    color: var(--text-color);
}

.clear-btn {
    border: none;
    background-color: var(--danger-color);
    color: white;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    transition: background-color 0.2s;
}

.clear-btn:hover {
    background-color: var(--danger-color-dark);
}

.history-panel ul {
    list-style: none;
    padding: 0;
    margin: 0;
    flex: 1;
    overflow-y: auto;
}

.history-panel li {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 4px;
    border-bottom: 1px solid var(--border-color);
    font-size: 15px;
}

.history-name {
    color: var(--dark-gray);
    font-weight: 500;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 60%;
}

.history-panel li:first-child .history-name {
    color: var(--text-color);
    font-weight: 700;
}

.history-date {
    font-size: 12px;
    color: #adb5bd;
    flex-shrink: 0;
    margin-left: 8px;
    /* Reduced margin to bring date closer */
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, Courier, monospace;
    /* Monospaced font */
    text-align: right;
}

.no-history {
    color: var(--dark-gray);
    text-align: center;
    margin-top: 30px;
    font-size: 15px;
}

.winner-overlay {
    position: absolute;
    inset: 0;
    z-index: 200;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    background: rgba(17, 24, 39, 0.42);
    backdrop-filter: blur(3px);
    box-sizing: border-box;
}

.winner-dialog {
    position: relative;
    width: min(420px, 100%);
    padding: 34px 32px 28px;
    border-radius: 8px;
    background: #ffffff;
    text-align: center;
    box-shadow: 0 24px 70px rgba(15, 23, 42, 0.28);
    animation: winner-pop 0.32s cubic-bezier(0.2, 0.9, 0.2, 1.08);
}

.winner-kicker {
    margin: 0 0 10px;
    color: var(--dark-gray);
    font-size: 16px;
    font-weight: 700;
}

.winner-dialog h2 {
    margin: 0;
    color: var(--primary-color);
    font-size: clamp(34px, 7vw, 52px);
    line-height: 1.15;
    overflow-wrap: anywhere;
}

.winner-close-btn {
    margin-top: 26px;
    border: none;
    border-radius: 6px;
    padding: 10px 26px;
    background: var(--primary-color);
    color: #ffffff;
    font-size: 15px;
    font-weight: 700;
    cursor: pointer;
    transition: background-color 0.2s, transform 0.2s;
}

.winner-close-btn:hover {
    background: var(--primary-color-dark);
    transform: translateY(-1px);
}

.fireworks {
    position: absolute;
    inset: 0;
    pointer-events: none;
    overflow: hidden;
}

.spark {
    --x: calc(((var(--i) * 47) % 100) * 1%);
    --y: calc((18 + ((var(--i) * 31) % 54)) * 1%);
    --angle: calc(var(--i) * 20deg);
    --distance: calc(42px + ((var(--i) % 5) * 14px));
    --delay: calc((var(--i) % 6) * 0.08s);
    position: absolute;
    left: var(--x);
    top: var(--y);
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: hsl(calc(var(--i) * 36), 82%, 58%);
    box-shadow:
        0 0 0 4px rgba(255, 255, 255, 0.18),
        0 0 18px currentColor;
    animation: spark-burst 1.05s ease-out var(--delay) both;
}

.winner-fade-enter-active,
.winner-fade-leave-active {
    transition: opacity 0.22s ease;
}

.winner-fade-enter-from,
.winner-fade-leave-to {
    opacity: 0;
}

@keyframes winner-pop {
    from {
        opacity: 0;
        transform: translateY(14px) scale(0.96);
    }

    to {
        opacity: 1;
        transform: translateY(0) scale(1);
    }
}

@keyframes spark-burst {
    0% {
        opacity: 0;
        transform: rotate(var(--angle)) translateX(0) scale(0.4);
    }

    20% {
        opacity: 1;
    }

    100% {
        opacity: 0;
        transform: rotate(var(--angle)) translateX(var(--distance)) scale(0.95);
    }
}

@media (max-width: 760px) {
    #app {
        flex-direction: column;
    }

    .sidebar {
        width: 100%;
        height: 36vh;
        min-height: 220px;
        border-right: none;
        border-bottom: 1px solid var(--border-color);
    }

    .main-content {
        flex: 1;
        padding: 24px;
    }

    .main-content.shifted-left {
        padding-right: 24px;
    }

    .right-panel {
        width: min(280px, 86vw);
    }
}
</style>
