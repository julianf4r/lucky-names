<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';

const props = defineProps<{
    prizes: string[];
    fontSizeMultiplier: number;
    textRadiusFactor: number;
    textWidthMultiplier: number;
}>();

const emit = defineEmits<{
    (e: 'winner-selected', winner: string): void;
}>();

const wheelContainerRef = ref<HTMLDivElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const rotation = ref(0);
const isSpinning = ref(false);

const colors = ['#E66060', '#46B9B0', '#3EA5BC', '#E5C25C', '#8B50A4', '#29B866'];
const primaryColor = ref('#007bff');

let resizeObserver: ResizeObserver | null = null;
let animationFrameId: number | null = null;
let activePrizesSnapshot: string[] | null = null;

const getActivePrizes = () => activePrizesSnapshot ?? props.prizes;

const getCanvasLogicalSize = (canvas: HTMLCanvasElement) => {
    const size = Math.min(canvas.clientWidth, canvas.clientHeight);
    return size > 0 ? size : canvas.width / window.devicePixelRatio;
};

const drawWheel = () => {
    const canvas = canvasRef.value;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const prizes = getActivePrizes();
    const size = getCanvasLogicalSize(canvas);
    const arc = Math.PI * 2 / (prizes.length || 1);
    const radius = size / 2;

    ctx.setTransform(window.devicePixelRatio, 0, 0, window.devicePixelRatio, 0, 0);
    ctx.clearRect(0, 0, size, size);

    // Outer border
    ctx.strokeStyle = '#E0E0E0';
    ctx.lineWidth = 10;
    ctx.beginPath();
    ctx.arc(radius, radius, radius - 5, 0, Math.PI * 2);
    ctx.stroke();

    ctx.save();
    ctx.translate(radius, radius);
    ctx.rotate(rotation.value);

    const prizeCount = prizes.length || 1;
    // Base font size calculation
    const baseFontSizeDivisor = 15 + Math.max(0, prizeCount - 8) * 1.0;
    const baseFontSize = Math.max(12, Math.floor(radius / baseFontSizeDivisor));

    // Apply user-controlled multipliers
    const fontSize = baseFontSize * props.fontSizeMultiplier;
    const textRadius = radius * props.textRadiusFactor;

    prizes.forEach((prize, i) => {
        ctx.beginPath();
        ctx.fillStyle = colors[i % colors.length] ?? '#007bff';
        ctx.moveTo(0, 0);
        ctx.arc(0, 0, radius - 10, arc * i - arc / 2, arc * (i + 1) - arc / 2);
        ctx.closePath();
        ctx.fill();

        ctx.save();
        ctx.fillStyle = '#fff';
        ctx.font = `bold ${fontSize}px Inter`;
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.rotate(arc * i);

        // Truncate text if too long
        const maxTextWidth = (radius - textRadius) * props.textWidthMultiplier;
        let prizeText = prize;
        if (ctx.measureText(prizeText).width > maxTextWidth) {
            while (ctx.measureText(prizeText + '...').width > maxTextWidth && prizeText.length > 0) {
                prizeText = prizeText.slice(0, -1);
            }
            prizeText += '...';
        }
        ctx.fillText(prizeText, textRadius, 0);
        ctx.restore();
    });

    ctx.restore();

    // Center button
    const centerButtonRadius = radius / 4;
    ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.arc(radius, radius, centerButtonRadius, 0, Math.PI * 2);
    ctx.closePath();
    ctx.fill();
    ctx.fillStyle = primaryColor.value;
    ctx.font = `bold ${Math.max(16, radius / 12)}px Inter`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(prizes.length > 0 ? '开始' : '暂无名单', radius, radius);

    // Draw pointer on the right, pointing outwards
    const pointerSize = Math.max(15, radius / 25);
    const pointerWidth = Math.max(40, radius / 10);
    ctx.fillStyle = primaryColor.value;
    ctx.beginPath();
    ctx.moveTo(radius * 2 - pointerWidth, radius);
    ctx.lineTo(radius * 2, radius - pointerSize);
    ctx.lineTo(radius * 2, radius + pointerSize);
    ctx.closePath();
    ctx.fill();
};

const getRandomUint32 = () => {
    const randomBuffer = new Uint32Array(1);
    crypto.getRandomValues(randomBuffer);
    return randomBuffer[0] ?? 0;
};

const getRandomInt = (max: number) => {
    const limit = Math.floor(0x100000000 / max) * max;
    let value = getRandomUint32();

    while (value >= limit) {
        value = getRandomUint32();
    }

    return value % max;
};

const getRandomFloat = () => {
    return getRandomUint32() / 0x100000000;
};

const spin = () => {
    const prizesSnapshot = props.prizes.slice();
    if (isSpinning.value || prizesSnapshot.length === 0) return;

    activePrizesSnapshot = prizesSnapshot;
    isSpinning.value = true;

    const winnerIndex = getRandomInt(prizesSnapshot.length);
    const totalRotations = 5;
    const arc = Math.PI * 2 / prizesSnapshot.length;
    // Align winner to the right side (angle 0)
    const targetRotation = (totalRotations * Math.PI * 2) - (winnerIndex * arc) + (getRandomFloat() * arc * 0.8 - arc * 0.4);

    let start: number | null = null;
    const duration = 5000; // 5 seconds
    const startRotation = rotation.value;

    const animate = (timestamp: number) => {
        if (!start) start = timestamp;
        const progress = timestamp - start;
        const easedProgress = 1 - Math.pow(1 - (progress / duration), 4); // easeOutQuart

        rotation.value = startRotation + (targetRotation - startRotation) * easedProgress;

        drawWheel();

        if (progress < duration) {
            animationFrameId = requestAnimationFrame(animate);
        } else {
            rotation.value = targetRotation % (Math.PI * 2);
            drawWheel();
            isSpinning.value = false;
            emit('winner-selected', prizesSnapshot[winnerIndex] ?? '');
            activePrizesSnapshot = null;
            animationFrameId = null;
        }
    };

    animationFrameId = requestAnimationFrame(animate);
};

const resizeCanvas = () => {
    const container = wheelContainerRef.value;
    const canvas = canvasRef.value;
    if (container && canvas) {
        const size = Math.min(container.clientWidth, container.clientHeight);
        const dpr = window.devicePixelRatio || 1;

        canvas.style.width = `${size}px`;
        canvas.style.height = `${size}px`;
        canvas.width = Math.floor(size * dpr);
        canvas.height = Math.floor(size * dpr);
        drawWheel();
    }
};

onMounted(() => {
    const rootStyle = getComputedStyle(document.documentElement);
    primaryColor.value = rootStyle.getPropertyValue('--primary-color').trim() || '#007bff';

    if (wheelContainerRef.value) {
        resizeObserver = new ResizeObserver(resizeCanvas);
        resizeObserver.observe(wheelContainerRef.value);
    }
    resizeCanvas();
});

onUnmounted(() => {
    if (animationFrameId !== null) {
        cancelAnimationFrame(animationFrameId);
    }

    resizeObserver?.disconnect();
});

watch(() => props.prizes, () => {
    if (!isSpinning.value) {
        rotation.value = 0;
        drawWheel();
    }
}, { deep: true });

watch(
    [() => props.fontSizeMultiplier, () => props.textRadiusFactor, () => props.textWidthMultiplier],
    () => {
        if (!isSpinning.value) {
            drawWheel();
        }
    }
);

</script>

<template>
    <div ref="wheelContainerRef" class="wheel-container">
        <canvas ref="canvasRef" :class="{ 'is-empty': prizes.length === 0 }" @click="spin"></canvas>
    </div>
</template>

<style scoped>
.wheel-container {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
}

canvas {
    cursor: pointer;
    border-radius: 50%;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.07);
}

canvas.is-empty {
    cursor: default;
}
</style>
