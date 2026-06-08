<template>
    <div class="space-y-6 max-w-2xl">
        <UBadge
            color="neutral"
            variant="outline"
            icon="i-lucide-info"
            :ui="{ base: 'w-full flex-col items-start gap-2 p-4' }"
            role="note"
            aria-label="How to use this tool">
            <div class="w-full space-y-2 text-left">
                <h3 class="text-sm font-semibold">How to use</h3>
                <ol class="list-decimal space-y-2 pl-4 text-sm">
                    <li>When prompted, allow microphone access — this site needs permission to read audio from your microphone.</li>
                    <li>Select your microphone and audio output device (headphones or speakers).</li>
                    <li>Place the output device close enough for the microphone to pick up the sound clearly.</li>
                    <li>
                        Click
                        <span class="font-medium">Preview</span>
                        to monitor the live microphone level.
                    </li>
                    <li>
                        Adjust the detection threshold: drag the red thumb so the level bar stays below it during silence, but crosses it when a click
                        sound is played.
                    </li>
                    <li>
                        Click
                        <span class="font-medium">Start</span>
                        . The tool plays random click sounds through the output device and records when the microphone detects each one.
                    </li>
                    <li>Latency is calculated as the time difference between playback and detection, in milliseconds.</li>
                    <li>
                        Click
                        <span class="font-medium">Stop</span>
                        when you are done. Review individual measurements and the average latency in the results section.
                    </li>
                </ol>
            </div>
        </UBadge>

        <div
            v-if="!supportsSetSinkId"
            class="rounded-md border border-red-300 bg-red-50 p-4 text-red-900 dark:border-red-700 dark:bg-red-950 dark:text-red-200"
            role="alert">
            <p class="text-sm">
                Your browser does not support output device selection. Audio will play through the system default output. Use Chrome or Edge for the
                most accurate results.
            </p>
        </div>

        <div
            v-if="errorMessage"
            class="rounded-md border border-red-300 bg-red-50 p-4 text-red-900 dark:border-red-700 dark:bg-red-950 dark:text-red-200"
            role="alert">
            <p class="text-sm">{{ errorMessage }}</p>
        </div>

        <div class="space-y-4 rounded-md border border-gray-200 bg-white p-4 dark:border-gray-600 dark:bg-slate-800">
            <div class="space-y-2">
                <label for="latency-input-device" class="block text-sm font-medium text-gray-700 dark:text-slate-300">
                    Audio Input Device (Microphone)
                </label>
                <USelect
                    id="latency-input-device"
                    v-model="selectedInput"
                    value-key="value"
                    :items="inputOptions"
                    color="neutral"
                    class="w-full"
                    :disabled="isActive"
                    aria-label="Select audio input device" />
            </div>

            <div class="space-y-2">
                <label for="latency-output-device" class="block text-sm font-medium text-gray-700 dark:text-slate-300">
                    Audio Output Device (Headphones or Speakers)
                </label>
                <USelect
                    id="latency-output-device"
                    v-model="selectedOutput"
                    value-key="value"
                    :items="outputOptions"
                    color="neutral"
                    class="w-full"
                    :disabled="isActive || !supportsSetSinkId"
                    aria-label="Select audio output device" />
            </div>

            <div class="space-y-2">
                <div class="flex items-center justify-between gap-2">
                    <label for="latency-threshold-meter" class="text-sm font-medium text-gray-700 dark:text-slate-300">Detection Threshold</label>
                    <div class="flex items-center gap-3">
                        <span class="text-xs text-gray-500 dark:text-slate-400">
                            Level
                            <span
                                class="ml-1 font-medium tabular-nums"
                                :class="isAboveThreshold ? 'text-green-600 dark:text-green-400' : 'text-gray-700 dark:text-slate-300'">
                                {{ currentMicLevel }}
                            </span>
                        </span>
                        <span class="text-xs text-gray-500 dark:text-slate-400">
                            Threshold
                            <span class="ml-1 font-medium tabular-nums text-gray-700 dark:text-slate-300">
                                {{ threshold }}
                            </span>
                        </span>
                        <UButton
                            v-if="!isMonitoring && !isActive"
                            label="Preview"
                            color="neutral"
                            variant="soft"
                            size="xs"
                            :disabled="!selectedInput"
                            aria-label="Start microphone preview"
                            @click="handleStartPreview" />
                        <UButton
                            v-else-if="isPreviewing && !isActive"
                            label="Stop"
                            color="neutral"
                            variant="ghost"
                            size="xs"
                            aria-label="Stop microphone preview"
                            @click="handleStopPreview" />
                    </div>
                </div>

                <div class="relative">
                    <div
                        class="pointer-events-none absolute inset-x-0 top-1/2 h-4 -translate-y-1/2 overflow-hidden rounded-full bg-gray-200 dark:bg-slate-600"
                        role="meter"
                        aria-label="Microphone audio level"
                        :aria-valuenow="currentMicLevel"
                        aria-valuemin="0"
                        :aria-valuemax="PEAK_MAX">
                        <div
                            class="h-full rounded-full transition-[width] duration-75"
                            :class="isAboveThreshold ? 'bg-green-500 dark:bg-green-400' : 'bg-blue-500 dark:bg-blue-400'"
                            :style="{ width: `${micLevelPercent}%` }" />
                    </div>

                    <USlider
                        v-model="threshold"
                        :min="THRESHOLD_MIN"
                        :max="PEAK_MAX"
                        :step="1"
                        :disabled="isActive"
                        :tooltip="true"
                        color="error"
                        size="md"
                        aria-label="Detection threshold"
                        :ui="{
                            track: 'bg-transparent dark:bg-transparent',
                            range: 'bg-transparent dark:bg-transparent',
                        }"
                        @pointerdown="handleThresholdInteraction"
                        @focus="handleThresholdInteraction" />
                </div>

                <p class="text-xs text-gray-500 dark:text-slate-400">
                    The blue/green bar shows microphone level; the red thumb shows the detection threshold. Set the threshold between silence and the
                    click sound.
                </p>
            </div>
        </div>

        <div class="flex flex-wrap items-center gap-4">
            <UButton
                v-if="!isActive"
                label="Start"
                color="primary"
                :loading="status === 'requesting'"
                :disabled="status === 'requesting' || inputOptions.length === 0"
                aria-label="Start latency measurement"
                @click="handleStart" />
            <UButton v-else label="Stop" color="neutral" variant="outline" aria-label="Stop latency measurement" @click="handleStop" />

            <UButton
                label="Refresh Devices"
                color="neutral"
                variant="ghost"
                :disabled="isActive"
                aria-label="Refresh audio devices"
                @click="handleRefreshDevices" />

            <p v-if="currentActionLabel" class="text-sm text-gray-600 dark:text-slate-400" role="status" aria-live="polite">
                {{ currentActionLabel }}
            </p>
        </div>

        <div v-if="measurements.length > 0" class="rounded-md border border-gray-200 bg-white p-4 dark:border-gray-600 dark:bg-slate-800">
            <div class="mb-4 flex items-center justify-between gap-2">
                <h3 class="text-sm font-semibold text-gray-900 dark:text-slate-200">Measurement Results</h3>
                <div class="flex items-center gap-1">
                    <UButton
                        label="Reset"
                        color="neutral"
                        variant="ghost"
                        size="xs"
                        :disabled="isActive"
                        aria-label="Reset measurements"
                        @click="handleResetMeasurements" />
                    <UButton
                        :label="showAdvanced ? 'Simple' : 'Advanced'"
                        color="neutral"
                        variant="ghost"
                        size="xs"
                        :aria-label="showAdvanced ? 'Switch to simple view' : 'Switch to advanced view'"
                        @click="showAdvanced = !showAdvanced" />
                </div>
            </div>

            <div v-if="!showAdvanced">
                <div v-if="latestMeasurement" class="flex flex-col items-center gap-1 py-4">
                    <span
                        class="text-4xl font-bold tabular-nums"
                        :class="latestMeasurement.latency === null ? 'text-amber-500 dark:text-amber-400' : 'text-gray-900 dark:text-slate-100'">
                        {{ latestMeasurement.latency === null ? '—' : `${latestMeasurement.latency.toFixed(1)}` }}
                    </span>
                    <span v-if="latestMeasurement.latency !== null" class="text-sm font-medium text-gray-500 dark:text-slate-400">ms</span>
                    <span v-else class="text-sm text-amber-500 dark:text-amber-400">Missed</span>
                    <span class="mt-1 text-xs text-gray-400 dark:text-slate-500">measurement #{{ latestMeasurement.id }}</span>
                </div>

                <div v-if="averageLatency !== null" class="mt-2 border-t border-gray-200 pt-3 text-center dark:border-gray-600">
                    <p class="text-sm text-gray-500 dark:text-slate-400">
                        Average
                        <span class="ml-1 font-semibold text-gray-900 dark:text-slate-200">{{ averageLatency.toFixed(1) }} ms</span>
                        <span class="ml-1 text-gray-400 dark:text-slate-500">({{ validMeasurementCount }}/{{ measurements.length }})</span>
                    </p>
                </div>
            </div>

            <div v-else>
                <ul class="space-y-1" role="list" aria-label="Latency measurement list">
                    <li
                        v-for="measurement in recentMeasurements"
                        :key="measurement.id"
                        class="flex items-center justify-between rounded-md px-2 py-1 text-sm odd:bg-gray-50 dark:odd:bg-slate-900">
                        <span class="text-gray-500 dark:text-slate-400">#{{ measurement.id }}</span>
                        <span
                            :class="
                                measurement.latency === null ? 'text-amber-600 dark:text-amber-400' : 'font-medium text-gray-900 dark:text-slate-200'
                            ">
                            {{ formatMeasurement(measurement) }}
                        </span>
                    </li>
                </ul>

                <div v-if="averageLatency !== null" class="mt-4 border-t border-gray-200 pt-3 dark:border-gray-600">
                    <p class="text-sm font-semibold text-gray-900 dark:text-slate-200">
                        Average: {{ averageLatency.toFixed(1) }} ms
                        <span class="font-normal text-gray-500 dark:text-slate-400">
                            ({{ validMeasurementCount }} / {{ measurements.length }} measurements)
                        </span>
                    </p>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
const MIN_DELAY_MS = 1000;
const MAX_DELAY_MS = 3000;
const DETECTION_TIMEOUT_MS = 500;
const CLICK_DURATION_SEC = 0.02;
const PEAK_MAX = 200;
const THRESHOLD_MIN = 5;
const MIC_GAIN = 4;

const status = ref('idle');
const currentAction = ref('idle');
const threshold = ref(25);
const selectedInput = ref('');
const selectedOutput = ref('');
const measurements = ref([]);
const errorMessage = ref('');
const currentMicLevel = ref(0);
const isPreviewing = ref(false);

const inputOptions = ref([]);
const outputOptions = ref([]);

let audioContext = null;
let analyserNode = null;
let micStream = null;
let previewAudioContext = null;
let previewAnalyserNode = null;
let previewMicStream = null;
let measurementLoopPromise = null;
let pendingTimeoutId = null;
let delayResolve = null;
let detectionFrameId = null;
let levelMonitorFrameId = null;
let levelMonitorAnalyser = null;
let measurementCounter = 0;

// Default true so SSR and initial client render match; actual value is set on mount.
const supportsSetSinkId = ref(true);

const isActive = computed(() => status.value === 'running' || status.value === 'requesting');

const isMonitoring = computed(() => isPreviewing.value || status.value === 'running');

const micLevelPercent = computed(() => Math.min(100, (currentMicLevel.value / PEAK_MAX) * 100));

const isAboveThreshold = computed(() => currentMicLevel.value >= threshold.value);

const currentActionLabel = computed(() => {
    const labels = {
        idle: '',
        waiting: 'Waiting for next measurement...',
        playing: 'Playing click sound...',
        detecting: 'Detecting sound...',
    };

    return labels[currentAction.value] ?? '';
});

const showAdvanced = ref(false);

const latestMeasurement = computed(() => measurements.value.at(-1) ?? null);

const recentMeasurements = computed(() => measurements.value.slice(-10).reverse());

const validMeasurements = computed(() => measurements.value.filter((measurement) => measurement.latency !== null));

const validMeasurementCount = computed(() => validMeasurements.value.length);

const averageLatency = computed(() => {
    if (validMeasurements.value.length === 0) {
        return null;
    }

    const total = validMeasurements.value.reduce((sum, measurement) => sum + measurement.latency, 0);
    return total / validMeasurements.value.length;
});

const formatMeasurement = (measurement) => {
    if (measurement.latency === null) {
        return 'Missed';
    }

    return `${measurement.latency.toFixed(1)} ms`;
};

const getDeviceLabel = (device, fallback) => {
    if (device.label) {
        return device.label;
    }

    return `${fallback} (${device.deviceId.slice(0, 8)}...)`;
};

const mapDeviceOptions = (devices, kind, fallback) => {
    return devices
        .filter((device) => device.kind === kind)
        .map((device) => ({
            label: getDeviceLabel(device, fallback),
            value: device.deviceId,
        }));
};

const setDefaultSelections = (inputs, outputs) => {
    if (!selectedInput.value || !inputs.some((option) => option.value === selectedInput.value)) {
        const defaultInput = inputs.find((option) => option.value === 'default') ?? inputs[0];
        selectedInput.value = defaultInput?.value ?? '';
    }

    if (!selectedOutput.value || !outputs.some((option) => option.value === selectedOutput.value)) {
        const defaultOutput = outputs.find((option) => option.value === 'default') ?? outputs[0];
        selectedOutput.value = defaultOutput?.value ?? '';
    }
};

const handleRefreshDevices = async () => {
    errorMessage.value = '';

    if (!navigator.mediaDevices?.enumerateDevices) {
        errorMessage.value = 'Your browser does not support listing audio devices.';
        return;
    }

    try {
        const devices = await navigator.mediaDevices.enumerateDevices();
        const inputs = mapDeviceOptions(devices, 'audioinput', 'Microphone');
        const outputs = mapDeviceOptions(devices, 'audiooutput', 'Speaker');

        inputOptions.value = inputs;
        outputOptions.value = outputs;
        setDefaultSelections(inputs, outputs);

        if (inputs.length === 0) {
            errorMessage.value = 'No audio input device found.';
        }
    } catch {
        errorMessage.value = 'An error occurred while listing audio devices.';
    }
};

const clearPendingTimeout = () => {
    if (pendingTimeoutId !== null) {
        clearTimeout(pendingTimeoutId);
        pendingTimeoutId = null;
    }

    if (delayResolve) {
        delayResolve();
        delayResolve = null;
    }
};

const cancelDetectionFrame = () => {
    if (detectionFrameId === null) {
        return;
    }

    cancelAnimationFrame(detectionFrameId);
    detectionFrameId = null;
};

const stopLevelMonitor = () => {
    if (levelMonitorFrameId !== null) {
        cancelAnimationFrame(levelMonitorFrameId);
        levelMonitorFrameId = null;
    }

    levelMonitorAnalyser = null;
    currentMicLevel.value = 0;
};

const startLevelMonitor = (analyser) => {
    stopLevelMonitor();

    levelMonitorAnalyser = analyser;
    const timeDomainData = new Uint8Array(analyser.fftSize);
    let displayLevel = 0;

    const tick = () => {
        if (!levelMonitorAnalyser) {
            return;
        }

        levelMonitorAnalyser.getByteTimeDomainData(timeDomainData);
        const peak = getPeakDeviation(timeDomainData);
        displayLevel = Math.max(peak, displayLevel * 0.88);
        currentMicLevel.value = Math.min(PEAK_MAX, Math.round(displayLevel));
        levelMonitorFrameId = requestAnimationFrame(tick);
    };

    levelMonitorFrameId = requestAnimationFrame(tick);
};

const getPeakDeviation = (data) => {
    let maxDeviation = 0;

    for (let index = 0; index < data.length; index++) {
        const deviation = Math.abs(data[index] - 128);
        if (deviation > maxDeviation) {
            maxDeviation = deviation;
        }
    }

    return maxDeviation;
};

const waitForRandomDelay = () => {
    const delayMs = MIN_DELAY_MS + Math.random() * (MAX_DELAY_MS - MIN_DELAY_MS);

    return new Promise((resolve) => {
        delayResolve = resolve;
        pendingTimeoutId = setTimeout(() => {
            pendingTimeoutId = null;
            delayResolve = null;
            resolve();
        }, delayMs);
    });
};

const createClickBuffer = (context) => {
    const sampleRate = context.sampleRate;
    const frameCount = Math.floor(sampleRate * CLICK_DURATION_SEC);
    const buffer = context.createBuffer(1, frameCount, sampleRate);
    const channelData = buffer.getChannelData(0);

    for (let index = 0; index < frameCount; index++) {
        const decay = Math.exp(-(index / frameCount) * 20);
        channelData[index] = (Math.random() * 2 - 1) * decay;
    }

    return buffer;
};

const playClick = async () => {
    if (!audioContext) {
        return;
    }

    if (audioContext.state === 'suspended') {
        await audioContext.resume();
    }

    const clickBuffer = createClickBuffer(audioContext);
    const source = audioContext.createBufferSource();
    const gainNode = audioContext.createGain();

    gainNode.gain.value = 1;
    source.buffer = clickBuffer;
    source.connect(gainNode);
    gainNode.connect(audioContext.destination);
    source.start();
};

const detectPeak = (playedAt) => {
    if (!analyserNode) {
        return Promise.resolve(null);
    }

    const timeDomainData = new Uint8Array(analyserNode.fftSize);

    return new Promise((resolve) => {
        const checkPeak = () => {
            if (status.value !== 'running') {
                cancelDetectionFrame();
                resolve(null);
                return;
            }

            analyserNode.getByteTimeDomainData(timeDomainData);
            const peakDeviation = getPeakDeviation(timeDomainData);
            const now = performance.now();

            if (peakDeviation >= threshold.value) {
                cancelDetectionFrame();
                resolve(now);
                return;
            }

            if (now - playedAt >= DETECTION_TIMEOUT_MS) {
                cancelDetectionFrame();
                resolve(null);
                return;
            }

            detectionFrameId = requestAnimationFrame(checkPeak);
        };

        detectionFrameId = requestAnimationFrame(checkPeak);
    });
};

const recordMeasurement = (latency) => {
    measurementCounter += 1;
    measurements.value.push({
        id: measurementCounter,
        latency,
    });
};

const handleResetMeasurements = () => {
    if (isActive.value) {
        return;
    }

    measurements.value = [];
    measurementCounter = 0;
    showAdvanced.value = false;
};

const runMeasurementLoop = async () => {
    while (status.value === 'running') {
        currentAction.value = 'waiting';
        await waitForRandomDelay();

        if (status.value !== 'running') {
            break;
        }

        currentAction.value = 'playing';
        const playedAt = performance.now();
        await playClick();

        currentAction.value = 'detecting';
        const detectedAt = await detectPeak(playedAt);

        if (status.value !== 'running') {
            break;
        }

        const latency = detectedAt === null ? null : detectedAt - playedAt;
        recordMeasurement(latency);
    }

    currentAction.value = 'idle';
};

const connectMicToAnalyser = (context, stream, analyser) => {
    const micSource = context.createMediaStreamSource(stream);
    const gainNode = context.createGain();

    gainNode.gain.value = MIC_GAIN;
    micSource.connect(gainNode);
    gainNode.connect(analyser);
};

const setupPreviewAudio = async () => {
    if (!selectedInput.value) {
        throw new Error('Please select a microphone.');
    }

    previewMicStream = await navigator.mediaDevices.getUserMedia({
        audio: {
            deviceId: { exact: selectedInput.value },
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false,
        },
    });

    previewAudioContext = new AudioContext();
    previewAnalyserNode = previewAudioContext.createAnalyser();
    previewAnalyserNode.fftSize = 2048;

    connectMicToAnalyser(previewAudioContext, previewMicStream, previewAnalyserNode);
    startLevelMonitor(previewAnalyserNode);
};

const cleanupPreviewAudio = async () => {
    stopLevelMonitor();

    if (previewMicStream) {
        previewMicStream.getTracks().forEach((track) => track.stop());
        previewMicStream = null;
    }

    if (previewAudioContext) {
        await previewAudioContext.close();
        previewAudioContext = null;
    }

    previewAnalyserNode = null;
    isPreviewing.value = false;
};

const handleStartPreview = async () => {
    if (isActive.value || isPreviewing.value) {
        return;
    }

    errorMessage.value = '';

    try {
        await setupPreviewAudio();
        isPreviewing.value = true;
    } catch (error) {
        await cleanupPreviewAudio();

        if (error instanceof DOMException && error.name === 'NotAllowedError') {
            errorMessage.value = 'Microphone permission denied. Please allow access in your browser settings.';
            return;
        }

        if (error instanceof Error) {
            errorMessage.value = error.message;
            return;
        }

        errorMessage.value = 'An error occurred while starting microphone preview.';
    }
};

const handleStopPreview = async () => {
    await cleanupPreviewAudio();
};

const handleThresholdInteraction = () => {
    if (!isActive.value && !isPreviewing.value && selectedInput.value) {
        handleStartPreview();
    }
};

const setupAudio = async () => {
    if (!selectedInput.value) {
        throw new Error('Please select a microphone.');
    }

    await handleStopPreview();

    micStream = await navigator.mediaDevices.getUserMedia({
        audio: {
            deviceId: { exact: selectedInput.value },
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false,
        },
    });

    audioContext = new AudioContext();
    analyserNode = audioContext.createAnalyser();
    analyserNode.fftSize = 2048;

    connectMicToAnalyser(audioContext, micStream, analyserNode);

    if (supportsSetSinkId.value && selectedOutput.value) {
        await audioContext.setSinkId(selectedOutput.value);
    }

    startLevelMonitor(analyserNode);
};

const cleanupAudio = async () => {
    clearPendingTimeout();
    cancelDetectionFrame();
    stopLevelMonitor();

    if (micStream) {
        micStream.getTracks().forEach((track) => track.stop());
        micStream = null;
    }

    if (audioContext) {
        await audioContext.close();
        audioContext = null;
    }

    analyserNode = null;
};

const handleStart = async () => {
    if (isActive.value) {
        return;
    }

    errorMessage.value = '';
    status.value = 'requesting';
    currentAction.value = 'idle';

    try {
        await setupAudio();
        status.value = 'running';
        measurementLoopPromise = runMeasurementLoop();
    } catch (error) {
        await cleanupAudio();
        status.value = 'idle';

        if (error instanceof DOMException && error.name === 'NotAllowedError') {
            errorMessage.value = 'Microphone permission denied. Please allow access in your browser settings.';
            return;
        }

        if (error instanceof DOMException && error.name === 'NotFoundError') {
            errorMessage.value = 'Selected microphone not found. Refresh devices and try again.';
            return;
        }

        if (error instanceof Error) {
            errorMessage.value = error.message;
            return;
        }

        errorMessage.value = 'An error occurred while starting the measurement.';
    }
};

const handleStop = async () => {
    if (!isActive.value) {
        return;
    }

    status.value = 'stopped';
    currentAction.value = 'idle';
    await cleanupAudio();

    if (measurementLoopPromise) {
        await measurementLoopPromise;
        measurementLoopPromise = null;
    }

    status.value = 'idle';
};

watch(selectedInput, async () => {
    if (!isPreviewing.value || isActive.value) {
        return;
    }

    await handleStopPreview();
    await handleStartPreview();
});

onMounted(async () => {
    supportsSetSinkId.value =
        typeof AudioContext !== 'undefined' && typeof AudioContext.prototype.setSinkId === 'function';

    await handleRefreshDevices();

    navigator.mediaDevices?.addEventListener('devicechange', handleRefreshDevices);
});

onUnmounted(async () => {
    navigator.mediaDevices?.removeEventListener('devicechange', handleRefreshDevices);
    await handleStop();
    await handleStopPreview();
});
</script>
