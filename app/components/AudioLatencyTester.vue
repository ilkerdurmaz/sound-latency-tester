<template>
    <div class="space-y-6 max-w-2xl">
        <UCollapsible
            v-model:open="showHowToUse"
            :unmount-on-hide="false"
            class="w-full rounded-md border border-gray-200 overflow-hidden dark:border-gray-600">
            <button
                type="button"
                class="flex w-full cursor-pointer items-center gap-2 rounded-md bg-gray-50 p-4 text-left dark:bg-slate-800/50"
                :aria-expanded="showHowToUse"
                aria-controls="how-to-use-content"
                :aria-label="t('tester.instructions.toggle')">
                <UIcon name="i-lucide-info" class="size-4 shrink-0 text-gray-500 dark:text-slate-400" aria-hidden="true" />
                <h3 class="flex-1 text-sm font-semibold text-gray-900 dark:text-slate-200">{{ t('tester.instructions.title') }}</h3>
                <UIcon
                    name="i-lucide-chevron-down"
                    class="size-4 shrink-0 text-gray-500 transition-transform duration-200 dark:text-slate-400"
                    :class="{ 'rotate-180': showHowToUse }"
                    aria-hidden="true" />
            </button>

            <template #content>
                <div
                    id="how-to-use-content"
                    class="border-t-0 border-gray-200 bg-gray-50 px-4 pb-4 dark:border-gray-600 dark:bg-slate-800/50"
                    role="note">
                    <ol class="list-decimal space-y-2 pl-4 text-sm text-gray-700 dark:text-slate-300">
                        <li>{{ t('tester.instructions.selectDevices') }}</li>
                        <li>{{ t('tester.instructions.placeDevice') }}</li>
                        <li>{{ t('tester.instructions.preview', { action: t('tester.actions.preview') }) }}</li>
                        <li>{{ t('tester.instructions.adjustThreshold') }}</li>
                        <li>{{ t('tester.instructions.start', { action: t('tester.actions.start') }) }}</li>
                        <li>{{ t('tester.instructions.calculation') }}</li>
                        <li>{{ t('tester.instructions.stop', { action: t('tester.actions.stop') }) }}</li>
                    </ol>
                </div>
            </template>
        </UCollapsible>

        <div
            v-if="!supportsSetSinkId"
            class="rounded-md border border-red-300 bg-red-50 p-4 text-red-900 dark:border-red-700 dark:bg-red-950 dark:text-red-200"
            role="alert">
            <p class="text-sm">
                {{ t('tester.support.outputDeviceUnsupported') }}
            </p>
        </div>

        <div
            v-if="errorMessage"
            class="rounded-md border border-red-300 bg-red-50 p-4 text-red-900 dark:border-red-700 dark:bg-red-950 dark:text-red-200"
            role="alert">
            <p class="text-sm">{{ errorMessage }}</p>
        </div>

        <div
            v-if="!hasMicrophoneAccess"
            class="rounded-md border border-blue-300 bg-blue-50 p-4 text-blue-950 dark:border-blue-700 dark:bg-blue-950 dark:text-blue-100"
            role="region"
            :aria-label="t('tester.permission.region')">
            <div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
                <div class="space-y-1">
                    <h3 class="text-sm font-semibold">{{ t('tester.permission.title') }}</h3>
                    <p class="text-sm">
                        {{ t('tester.permission.description') }}
                    </p>
                    <p v-if="permissionDenied" class="text-sm text-red-700 dark:text-red-300">
                        {{ t('tester.permission.denied') }}
                    </p>
                </div>
                <UButton
                    :label="t('tester.permission.allow')"
                    icon="i-lucide-mic"
                    color="primary"
                    class="shrink-0"
                    :loading="isRequestingPermission"
                    :aria-label="t('tester.permission.allowAria')"
                    @click="handleRequestMicrophoneAccess" />
            </div>
        </div>

        <div v-if="measurements.length > 0" class="rounded-md border border-gray-200 bg-white p-4 dark:border-gray-600 dark:bg-slate-800">
            <div class="mb-4 flex items-center justify-between gap-2">
                <h3 class="text-sm font-semibold text-gray-900 dark:text-slate-200">{{ t('tester.results.title') }}</h3>
                <div class="flex items-center gap-1">
                    <UButton
                        :label="t('tester.results.reset')"
                        color="neutral"
                        variant="ghost"
                        size="xs"
                        :disabled="isActive"
                        :aria-label="t('tester.results.resetAria')"
                        @click="handleResetMeasurements" />
                    <UButton
                        :label="showAdvanced ? t('tester.results.simple') : t('tester.results.advanced')"
                        color="neutral"
                        variant="ghost"
                        size="xs"
                        :aria-label="showAdvanced ? t('tester.results.switchSimple') : t('tester.results.switchAdvanced')"
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
                    <span v-else class="text-sm text-amber-500 dark:text-amber-400">{{ t('tester.results.missed') }}</span>
                    <span class="mt-1 text-xs text-gray-400 dark:text-slate-500">{{ t('tester.results.measurementNumber', { id: latestMeasurement.id }) }}</span>
                </div>

                <div v-if="averageLatency !== null" class="mt-2 border-t border-gray-200 pt-3 text-center dark:border-gray-600">
                    <p class="text-sm text-gray-500 dark:text-slate-400">
                        {{ t('tester.results.average') }}
                        <span class="ml-1 font-semibold text-gray-900 dark:text-slate-200">{{ averageLatency.toFixed(1) }} ms</span>
                        <span class="ml-1 text-gray-400 dark:text-slate-500">
                            {{ t('tester.results.validCount', { valid: validMeasurementCount, total: measurements.length }) }}
                        </span>
                    </p>
                </div>
            </div>

            <div v-else>
                <ul class="space-y-1" role="list" :aria-label="t('tester.results.listAria')">
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
                        {{ t('tester.results.averageWithValue', { value: averageLatency.toFixed(1) }) }}
                        <span class="font-normal text-gray-500 dark:text-slate-400">
                            {{ t('tester.results.validMeasurements', { valid: validMeasurementCount, total: measurements.length }) }}
                        </span>
                    </p>
                </div>
            </div>
        </div>

        <div
            class="space-y-4 rounded-md border border-gray-200 bg-white p-4 dark:border-gray-600 dark:bg-slate-800"
            :class="{ 'pointer-events-none opacity-50': !hasMicrophoneAccess }"
            :aria-hidden="!hasMicrophoneAccess">
            <div class="space-y-2">
                <label for="latency-input-device" class="block text-sm font-medium text-gray-700 dark:text-slate-300">
                    {{ t('tester.devices.inputLabel') }}
                </label>
                <USelect
                    id="latency-input-device"
                    v-model="selectedInput"
                    value-key="value"
                    :items="inputOptions"
                    :placeholder="t('tester.devices.inputPlaceholder')"
                    color="neutral"
                    class="w-full"
                    :disabled="isAnyOperationActive"
                    :aria-label="t('tester.devices.inputAria')" />
            </div>

            <div class="space-y-2">
                <label for="latency-output-device" class="block text-sm font-medium text-gray-700 dark:text-slate-300">
                    {{ t('tester.devices.outputLabel') }}
                </label>
                <USelect
                    id="latency-output-device"
                    v-model="selectedOutput"
                    value-key="value"
                    :items="outputOptions"
                    :placeholder="t('tester.devices.outputPlaceholder')"
                    color="neutral"
                    class="w-full"
                    :disabled="isAnyOperationActive || !supportsSetSinkId"
                    :aria-label="t('tester.devices.outputAria')" />
            </div>

            <div class="space-y-2">
                <div class="flex items-center justify-between gap-2">
                    <label for="latency-threshold-meter" class="text-sm font-medium text-gray-700 dark:text-slate-300">{{ t('tester.threshold.label') }}</label>
                    <div class="flex items-center gap-3">
                        <span class="text-xs text-gray-500 dark:text-slate-400">
                            {{ t('tester.threshold.level') }}
                            <span
                                class="ml-1 font-medium tabular-nums"
                                :class="isAboveThreshold ? 'text-green-600 dark:text-green-400' : 'text-gray-700 dark:text-slate-300'">
                                {{ currentMicLevel }}
                            </span>
                        </span>
                        <span class="text-xs text-gray-500 dark:text-slate-400">
                            {{ t('tester.threshold.threshold') }}
                            <span class="ml-1 font-medium tabular-nums text-gray-700 dark:text-slate-300">
                                {{ threshold }}
                            </span>
                        </span>
                        <UButton
                            v-if="!isMonitoring && !isAnyOperationActive"
                            :label="t('tester.actions.preview')"
                            color="neutral"
                            variant="outline"
                            size="xs"
                            :disabled="isAnyOperationActive"
                            :aria-label="t('tester.actions.startPreviewAria')"
                            @click="handleStartPreview" />
                        <UButton
                            v-else-if="isPreviewing && !isActive"
                            :label="t('tester.actions.stop')"
                            color="neutral"
                            variant="ghost"
                            size="xs"
                            :aria-label="t('tester.actions.stopPreviewAria')"
                            @click="handleStopPreview" />
                    </div>
                </div>

                <div class="relative">
                    <div
                        class="pointer-events-none absolute inset-x-0 top-1/2 h-4 -translate-y-1/2 overflow-hidden rounded-full bg-gray-200 dark:bg-slate-600"
                        role="meter"
                        :aria-label="t('tester.threshold.meterAria')"
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
                        :disabled="isAnyOperationActive"
                        :tooltip="true"
                        color="error"
                        size="md"
                        :aria-label="t('tester.threshold.sliderAria')"
                        :ui="{
                            track: 'bg-transparent dark:bg-transparent',
                            range: 'bg-transparent dark:bg-transparent',
                        }"
                        @pointerdown="handleThresholdInteraction"
                        @focus="handleThresholdInteraction" />
                </div>

                <p class="text-xs text-gray-500 dark:text-slate-400">
                    {{ t('tester.threshold.description') }}
                </p>
            </div>
        </div>

        <div
            class="space-y-4 rounded-md border border-gray-200 bg-white p-4 dark:border-gray-600 dark:bg-slate-800"
            :class="{ 'pointer-events-none opacity-50': !hasMicrophoneAccess }"
            :aria-hidden="!hasMicrophoneAccess">
            <div class="flex flex-col gap-3 sm:flex-row sm:items-start sm:justify-between">
                <div class="space-y-1">
                    <h3 class="text-sm font-semibold text-gray-900 dark:text-slate-200">{{ t('tester.recording.title') }}</h3>
                    <p class="text-sm text-gray-500 dark:text-slate-400">{{ recordingStatusLabel }}</p>
                </div>

                <div class="flex flex-wrap items-center gap-2">
                    <UButton
                        v-if="!isRecordingActive"
                        :label="t('tester.actions.record')"
                        icon="i-lucide-circle"
                        color="primary"
                        size="sm"
                        :disabled="isActive || isPlaybackActive || !hasMicrophoneAccess"
                        :aria-label="t('tester.actions.startRecordingAria')"
                        @click="handleStartRecording" />
                    <UButton
                        v-else
                        :label="t('tester.actions.stopRecording')"
                        icon="i-lucide-square"
                        color="neutral"
                        variant="outline"
                        size="sm"
                        :disabled="recordingStatus === 'requesting' || recordingStatus === 'stopping'"
                        :aria-label="t('tester.actions.stopRecordingAria')"
                        @click="handleStopRecording" />

                    <UButton
                        v-if="!isPlaybackActive"
                        :label="t('tester.actions.play')"
                        icon="i-lucide-play"
                        color="neutral"
                        variant="outline"
                        size="sm"
                        :disabled="!recordedUrl || isActive || isRecordingActive"
                        :aria-label="t('tester.actions.playRecordingAria')"
                        @click="handleStartPlayback" />
                    <UButton
                        v-else
                        :label="t('tester.actions.stopPlayback')"
                        icon="i-lucide-stop-circle"
                        color="neutral"
                        variant="outline"
                        size="sm"
                        :aria-label="t('tester.actions.stopPlaybackAria')"
                        @click="handleStopPlayback" />

                    <UButton
                        :label="t('tester.actions.download')"
                        icon="i-lucide-download"
                        color="neutral"
                        variant="ghost"
                        size="sm"
                        :disabled="!recordedUrl || isAnyOperationActive"
                        :aria-label="t('tester.actions.downloadRecordingAria')"
                        @click="handleDownloadRecording" />
                </div>
            </div>

            <div class="space-y-2">
                <div class="flex items-center justify-between gap-2">
                    <label for="recording-duration" class="text-sm font-medium text-gray-700 dark:text-slate-300">{{ t('tester.recording.duration') }}</label>
                    <div class="flex items-center gap-2">
                        <UInput
                            id="recording-duration"
                            v-model.number="recordingDurationSeconds"
                            type="number"
                            :min="RECORDING_DURATION_MIN"
                            :max="RECORDING_DURATION_MAX"
                            :step="1"
                            size="xs"
                            class="w-20"
                            :disabled="isAnyOperationActive"
                            :aria-label="t('tester.recording.durationInputAria')"
                            @change="normalizeRecordingDuration" />
                        <span class="text-xs text-gray-500 dark:text-slate-400">{{ t('tester.recording.seconds') }}</span>
                    </div>
                </div>

                <USlider
                    v-model="recordingDurationSeconds"
                    :min="RECORDING_DURATION_MIN"
                    :max="RECORDING_DURATION_MAX"
                    :step="1"
                    :disabled="isAnyOperationActive"
                    color="primary"
                    size="sm"
                    :aria-label="t('tester.recording.durationSliderAria')" />

                <div
                    class="h-2 overflow-hidden rounded-full bg-gray-200 dark:bg-slate-600"
                    role="meter"
                    :aria-label="t('tester.recording.progressAria')"
                    :aria-valuenow="recordingProgressPercent"
                    aria-valuemin="0"
                    aria-valuemax="100">
                    <div class="h-full rounded-full bg-green-500 transition-[width] duration-150 dark:bg-green-400" :style="{ width: `${recordingProgressPercent}%` }" />
                </div>
            </div>

            <p v-if="recordedUrl" class="text-xs text-gray-500 dark:text-slate-400">
                {{ t('tester.recording.lastRecording', { fileName: recordingFileName }) }}
            </p>
        </div>

        <div class="flex flex-wrap items-center gap-4" :class="{ 'pointer-events-none opacity-50': !hasMicrophoneAccess }">
            <UButton
                v-if="!isActive"
                :label="t('tester.actions.start')"
                color="primary"
                :loading="status === 'requesting'"
                :disabled="status === 'requesting' || !hasMicrophoneAccess || isRecordingBusy"
                :aria-label="t('tester.actions.startMeasurementAria')"
                @click="handleStart" />
            <UButton v-else :label="t('tester.actions.stop')" color="neutral" variant="outline" :aria-label="t('tester.actions.stopMeasurementAria')" @click="handleStop" />

            <UButton
                :label="t('tester.actions.refreshDevices')"
                color="neutral"
                variant="ghost"
                :disabled="isAnyOperationActive"
                :aria-label="t('tester.actions.refreshDevicesAria')"
                @click="handleRefreshDevices(true)" />

            <p v-if="currentActionLabel" class="text-sm text-gray-600 dark:text-slate-400" role="status" aria-live="polite">
                {{ currentActionLabel }}
            </p>
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
const RECORDING_DURATION_MIN = 1;
const RECORDING_DURATION_MAX = 30;
const RECORDING_DURATION_DEFAULT = 30;

const { locale, t } = useI18n();

const status = ref('idle');
const currentAction = ref('idle');
const threshold = ref(25);
const selectedInput = ref(undefined);
const selectedOutput = ref(undefined);
const measurements = ref([]);
const errorMessageKey = ref('');
const errorMessageParams = ref({});
const currentMicLevel = ref(0);
const isPreviewing = ref(false);
const hasMicrophoneAccess = ref(false);
const isRequestingPermission = ref(false);
const permissionDenied = ref(false);
const recordingDurationSeconds = ref(RECORDING_DURATION_DEFAULT);
const recordingStatus = ref('idle');
const recordedBlob = ref(null);
const recordedUrl = ref('');
const recordedMimeType = ref('');
const recordedAt = ref(null);
const recordingElapsedSeconds = ref(0);

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
let mediaRecorder = null;
let recordingStream = null;
let recordingStopTimeoutId = null;
let recordingProgressIntervalId = null;
let recordingStartedAt = 0;
let recordingStopPromise = null;
let playbackAudio = null;

// Default true so SSR and initial client render match; actual value is set on mount.
const supportsSetSinkId = ref(true);

const isActive = computed(() => status.value === 'running' || status.value === 'requesting');

const isRecordingActive = computed(() => ['requesting', 'recording', 'stopping'].includes(recordingStatus.value));

const isPlaybackActive = computed(() => recordingStatus.value === 'playing');

const isRecordingBusy = computed(() => isRecordingActive.value || isPlaybackActive.value);

const isAnyOperationActive = computed(() => isActive.value || isRecordingBusy.value);

const isMonitoring = computed(() => isPreviewing.value || status.value === 'running');

const micLevelPercent = computed(() => Math.min(100, (currentMicLevel.value / PEAK_MAX) * 100));

const isAboveThreshold = computed(() => currentMicLevel.value >= threshold.value);

const errorMessage = computed(() => {
    if (!errorMessageKey.value) {
        return '';
    }

    return t(errorMessageKey.value, errorMessageParams.value);
});

const setErrorMessage = (key, params = {}) => {
    errorMessageKey.value = key;
    errorMessageParams.value = params;
};

const setNativeErrorMessage = (error) => {
    setErrorMessage('tester.errors.native', { message: error.message });
};

const clearErrorMessage = () => {
    errorMessageKey.value = '';
    errorMessageParams.value = {};
};

const currentActionLabel = computed(() => {
    const labels = {
        idle: '',
        waiting: t('tester.status.waiting'),
        playing: t('tester.status.playing'),
        detecting: t('tester.status.detecting'),
    };

    return labels[currentAction.value] ?? '';
});

const showAdvanced = ref(false);
const showHowToUse = ref(true);

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

const recordingProgressPercent = computed(() => {
    if (recordingStatus.value !== 'recording') {
        return recordedBlob.value ? 100 : 0;
    }

    return Math.min(100, (recordingElapsedSeconds.value / recordingDurationSeconds.value) * 100);
});

const recordingStatusLabel = computed(() => {
    if (recordingStatus.value === 'requesting') {
        return t('tester.recording.status.starting');
    }

    if (recordingStatus.value === 'recording') {
        return t('tester.recording.status.recording', {
            elapsed: recordingElapsedSeconds.value,
            duration: recordingDurationSeconds.value,
        });
    }

    if (recordingStatus.value === 'stopping') {
        return t('tester.recording.status.saving');
    }

    if (recordingStatus.value === 'playing') {
        return t('tester.recording.status.playing');
    }

    if (recordedBlob.value) {
        return t('tester.recording.status.ready');
    }

    return t('tester.recording.status.empty');
});

const recordingFileName = computed(() => {
    const createdAt = recordedAt.value ?? new Date();
    const pad = (value) => String(value).padStart(2, '0');
    const datePart = `${createdAt.getFullYear()}${pad(createdAt.getMonth() + 1)}${pad(createdAt.getDate())}`;
    const timePart = `${pad(createdAt.getHours())}${pad(createdAt.getMinutes())}${pad(createdAt.getSeconds())}`;
    const extension = getRecordingExtension(recordedMimeType.value);

    return `audio-recording-${datePart}-${timePart}.${extension}`;
});

const formatMeasurement = (measurement) => {
    if (measurement.latency === null) {
        return t('tester.results.missed');
    }

    return t('tester.results.latencyValue', { value: measurement.latency.toFixed(1) });
};

const getRecordingExtension = (mimeType) => {
    if (mimeType.includes('ogg')) {
        return 'ogg';
    }

    if (mimeType.includes('mp4')) {
        return 'mp4';
    }

    if (mimeType.includes('mpeg')) {
        return 'mp3';
    }

    if (mimeType.includes('wav')) {
        return 'wav';
    }

    return 'webm';
};

const normalizeRecordingDuration = () => {
    const numericDuration = Number(recordingDurationSeconds.value);

    if (!Number.isFinite(numericDuration)) {
        recordingDurationSeconds.value = RECORDING_DURATION_DEFAULT;
        return;
    }

    recordingDurationSeconds.value = Math.min(RECORDING_DURATION_MAX, Math.max(RECORDING_DURATION_MIN, Math.round(numericDuration)));
};

const clearRecordedAudio = () => {
    if (recordedUrl.value) {
        URL.revokeObjectURL(recordedUrl.value);
    }

    recordedBlob.value = null;
    recordedUrl.value = '';
    recordedMimeType.value = '';
    recordedAt.value = null;
};

const clearRecordingTimers = () => {
    if (recordingStopTimeoutId !== null) {
        clearTimeout(recordingStopTimeoutId);
        recordingStopTimeoutId = null;
    }

    if (recordingProgressIntervalId !== null) {
        clearInterval(recordingProgressIntervalId);
        recordingProgressIntervalId = null;
    }
};

const cleanupRecordingStream = () => {
    if (recordingStream) {
        recordingStream.getTracks().forEach((track) => track.stop());
        recordingStream = null;
    }
};

const updateRecordingElapsed = () => {
    if (recordingStartedAt === 0) {
        recordingElapsedSeconds.value = 0;
        return;
    }

    const elapsed = Math.ceil((performance.now() - recordingStartedAt) / 1000);
    recordingElapsedSeconds.value = Math.min(recordingDurationSeconds.value, Math.max(0, elapsed));
};

const setupRecordingStopPromise = (recorder, chunks) => {
    recordingStopPromise = new Promise((resolve) => {
        recorder.addEventListener(
            'stop',
            () => {
                clearRecordingTimers();
                updateRecordingElapsed();
                cleanupRecordingStream();

                if (chunks.length > 0) {
                    const mimeType = recorder.mimeType || chunks[0].type || '';
                    const blob = new Blob(chunks, { type: mimeType });

                    recordedBlob.value = blob;
                    recordedMimeType.value = blob.type || mimeType;
                    recordedAt.value = new Date();
                    recordedUrl.value = URL.createObjectURL(blob);
                }

                mediaRecorder = null;
                recordingStopPromise = null;
                recordingStartedAt = 0;
                recordingStatus.value = 'idle';
                resolve();
            },
            { once: true }
        );
    });
};

const cleanupPlaybackAudio = () => {
    if (!playbackAudio) {
        return;
    }

    playbackAudio.pause();
    playbackAudio.removeAttribute('src');
    playbackAudio.load();
    playbackAudio = null;
};

const getDeviceLabel = (device, fallback) => {
    if (device.label) {
        return device.label;
    }

    return `${fallback} (${device.deviceId.slice(0, 8)}...)`;
};

const mapDeviceOptions = (devices, kind, fallback) => {
    return devices
        .filter((device) => device.kind === kind && device.deviceId !== '')
        .map((device) => ({
            label: getDeviceLabel(device, fallback),
            value: device.deviceId,
        }));
};

const setDefaultSelections = (inputs, outputs) => {
    if (!selectedInput.value || !inputs.some((option) => option.value === selectedInput.value)) {
        const defaultInput = inputs.find((option) => option.value === 'default') ?? inputs[0];
        selectedInput.value = defaultInput?.value;
    }

    if (!selectedOutput.value || !outputs.some((option) => option.value === selectedOutput.value)) {
        const defaultOutput = outputs.find((option) => option.value === 'default') ?? outputs[0];
        selectedOutput.value = defaultOutput?.value;
    }
};

const handleRefreshDevices = async (requestPermission = false) => {
    if (requestPermission) {
        clearErrorMessage();
        permissionDenied.value = false;
    }

    if (!navigator.mediaDevices?.enumerateDevices) {
        setErrorMessage('tester.errors.listDevicesUnsupported');
        hasMicrophoneAccess.value = false;
        return false;
    }

    try {
        if (requestPermission && navigator.mediaDevices.getUserMedia) {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                stream.getTracks().forEach((track) => track.stop());
            } catch (permissionError) {
                hasMicrophoneAccess.value = false;

                if (permissionError instanceof DOMException && permissionError.name === 'NotAllowedError') {
                    permissionDenied.value = true;
                    setErrorMessage('tester.errors.microphonePermissionDenied');
                    return false;
                }
            }
        }

        const devices = await navigator.mediaDevices.enumerateDevices();
        const inputs = mapDeviceOptions(devices, 'audioinput', t('tester.devices.microphoneFallback'));
        const outputs = mapDeviceOptions(devices, 'audiooutput', t('tester.devices.speakerFallback'));

        inputOptions.value = inputs;
        outputOptions.value = outputs;
        setDefaultSelections(inputs, outputs);
        hasMicrophoneAccess.value = inputs.length > 0;

        if (inputs.length === 0 && requestPermission) {
            setErrorMessage('tester.errors.noAudioInput');
            return false;
        }

        return hasMicrophoneAccess.value;
    } catch {
        setErrorMessage('tester.errors.listDevicesFailed');
        hasMicrophoneAccess.value = false;
        return false;
    }
};

const handleRequestMicrophoneAccess = async () => {
    isRequestingPermission.value = true;

    try {
        await handleRefreshDevices(true);
    } finally {
        isRequestingPermission.value = false;
    }
};

const handleDeviceChange = () => handleRefreshDevices(false);

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
    previewMicStream = await navigator.mediaDevices.getUserMedia({
        audio: {
            deviceId: selectedInput.value ? { exact: selectedInput.value } : undefined,
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false,
        },
    });

    await handleRefreshDevices();

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
    if (isAnyOperationActive.value || isPreviewing.value) {
        return;
    }

    clearErrorMessage();

    try {
        await setupPreviewAudio();
        isPreviewing.value = true;
    } catch (error) {
        await cleanupPreviewAudio();

        if (error instanceof DOMException && error.name === 'NotAllowedError') {
            setErrorMessage('tester.errors.microphonePermissionDenied');
            return;
        }

        if (error instanceof Error) {
            setNativeErrorMessage(error);
            return;
        }

        setErrorMessage('tester.errors.previewFailed');
    }
};

const handleStopPreview = async () => {
    await cleanupPreviewAudio();
};

const handleThresholdInteraction = () => {
    if (!isAnyOperationActive.value && !isPreviewing.value) {
        handleStartPreview();
    }
};

const handleStartRecording = async () => {
    if (isActive.value || isRecordingBusy.value) {
        return;
    }

    if (typeof MediaRecorder === 'undefined') {
        setErrorMessage('tester.errors.recordingUnsupported');
        return;
    }

    clearErrorMessage();
    normalizeRecordingDuration();
    recordingStatus.value = 'requesting';
    recordingElapsedSeconds.value = 0;

    try {
        await handleStopPreview();

        recordingStream = await navigator.mediaDevices.getUserMedia({
            audio: {
                deviceId: selectedInput.value ? { exact: selectedInput.value } : undefined,
                echoCancellation: false,
                noiseSuppression: false,
                autoGainControl: false,
            },
        });

        await handleRefreshDevices();

        const chunks = [];
        mediaRecorder = new MediaRecorder(recordingStream);

        mediaRecorder.addEventListener('dataavailable', (event) => {
            if (event.data.size > 0) {
                chunks.push(event.data);
            }
        });

        mediaRecorder.addEventListener(
            'error',
            () => {
                setErrorMessage('tester.errors.recordingFailed');
                handleStopRecording();
            },
            { once: true }
        );

        setupRecordingStopPromise(mediaRecorder, chunks);
        mediaRecorder.start();
        clearRecordedAudio();

        recordingStartedAt = performance.now();
        updateRecordingElapsed();
        recordingProgressIntervalId = setInterval(updateRecordingElapsed, 250);
        recordingStopTimeoutId = setTimeout(() => {
            handleStopRecording();
        }, recordingDurationSeconds.value * 1000);

        recordingStatus.value = 'recording';
    } catch (error) {
        clearRecordingTimers();
        cleanupRecordingStream();
        mediaRecorder = null;
        recordingStopPromise = null;
        recordingStartedAt = 0;
        recordingStatus.value = 'idle';

        if (error instanceof DOMException && error.name === 'NotAllowedError') {
            setErrorMessage('tester.errors.microphonePermissionDenied');
            return;
        }

        if (error instanceof DOMException && error.name === 'NotFoundError') {
            setErrorMessage('tester.errors.selectedMicrophoneMissing');
            return;
        }

        if (error instanceof Error) {
            setNativeErrorMessage(error);
            return;
        }

        setErrorMessage('tester.errors.recordingStartFailed');
    }
};

const handleStopRecording = async () => {
    if (!isRecordingActive.value) {
        return;
    }

    clearRecordingTimers();
    recordingStatus.value = 'stopping';

    if (mediaRecorder && mediaRecorder.state !== 'inactive') {
        try {
            mediaRecorder.stop();
        } catch {
            cleanupRecordingStream();
            mediaRecorder = null;
            recordingStopPromise = null;
            recordingStartedAt = 0;
            recordingStatus.value = 'idle';
            return;
        }
    }

    if (recordingStopPromise) {
        await recordingStopPromise;
        return;
    }

    cleanupRecordingStream();
    mediaRecorder = null;
    recordingStartedAt = 0;
    recordingStatus.value = 'idle';
};

const handleStartPlayback = async () => {
    if (!recordedUrl.value || isActive.value || isRecordingBusy.value) {
        return;
    }

    clearErrorMessage();

    try {
        await handleStopPreview();

        recordingStatus.value = 'playing';
        playbackAudio = new Audio(recordedUrl.value);

        playbackAudio.addEventListener(
            'ended',
            () => {
                cleanupPlaybackAudio();
                recordingStatus.value = 'idle';
            },
            { once: true }
        );

        playbackAudio.addEventListener(
            'error',
            () => {
                cleanupPlaybackAudio();
                recordingStatus.value = 'idle';
                setErrorMessage('tester.errors.playbackFailed');
            },
            { once: true }
        );

        if (typeof playbackAudio.setSinkId === 'function' && selectedOutput.value) {
            try {
                await playbackAudio.setSinkId(selectedOutput.value);
            } catch {
                setErrorMessage('tester.errors.selectedOutputUnavailable');
            }
        }

        await playbackAudio.play();
    } catch (error) {
        cleanupPlaybackAudio();
        recordingStatus.value = 'idle';

        if (error instanceof Error) {
            setNativeErrorMessage(error);
            return;
        }

        setErrorMessage('tester.errors.playbackFailed');
    }
};

const handleStopPlayback = () => {
    if (!isPlaybackActive.value) {
        return;
    }

    cleanupPlaybackAudio();
    recordingStatus.value = 'idle';
};

const handleDownloadRecording = () => {
    if (!recordedUrl.value) {
        return;
    }

    const anchor = document.createElement('a');
    anchor.href = recordedUrl.value;
    anchor.download = recordingFileName.value;
    document.body.appendChild(anchor);
    anchor.click();
    anchor.remove();
};

const setupAudio = async () => {
    await handleStopPreview();

    micStream = await navigator.mediaDevices.getUserMedia({
        audio: {
            deviceId: selectedInput.value ? { exact: selectedInput.value } : undefined,
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false,
        },
    });

    await handleRefreshDevices();

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
    if (isActive.value || isRecordingBusy.value) {
        return;
    }

    clearErrorMessage();
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
            setErrorMessage('tester.errors.microphonePermissionDenied');
            return;
        }

        if (error instanceof DOMException && error.name === 'NotFoundError') {
            setErrorMessage('tester.errors.selectedMicrophoneMissing');
            return;
        }

        if (error instanceof Error) {
            setNativeErrorMessage(error);
            return;
        }

        setErrorMessage('tester.errors.measurementStartFailed');
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

watch(isActive, (active) => {
    if (active) {
        showHowToUse.value = false;
    }
});

watch(selectedInput, async () => {
    if (!isPreviewing.value || isActive.value) {
        return;
    }

    await handleStopPreview();
    await handleStartPreview();
});

watch(recordingDurationSeconds, () => {
    if (!isRecordingActive.value) {
        normalizeRecordingDuration();
    }
});

watch(locale, () => {
    handleRefreshDevices(false);
});

onMounted(async () => {
    supportsSetSinkId.value = typeof AudioContext !== 'undefined' && typeof AudioContext.prototype.setSinkId === 'function';

    await handleRefreshDevices();

    navigator.mediaDevices?.addEventListener('devicechange', handleDeviceChange);
});

onUnmounted(async () => {
    navigator.mediaDevices?.removeEventListener('devicechange', handleDeviceChange);
    await handleStop();
    await handleStopPreview();
    handleStopPlayback();
    await handleStopRecording();
    clearRecordedAudio();
});
</script>
