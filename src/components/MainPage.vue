<script setup lang="ts">
import { onMounted, ref } from 'vue';
// @ts-ignore
import RecordRTC from 'recordrtc';

const live = ref<HTMLVideoElement | null>(null);
const preview = ref<HTMLVideoElement | null>(null);
const recordingStream = ref<MediaStream | null>(null);
const mediaRecorder = ref<MediaRecorder | null>(null);
const totalChunks = ref<Blob[]>([])
const isRecording = ref<Boolean>(false);
const isPausing = ref<Boolean>(false);
const finalBlob = ref<Blob | null>(null);
const isCompletedStatus = ref<Boolean>(false);
const recorderType = ref<{ type: number, value: string }[]>([{ type: 1, value: 'In-built Recorder' }, { type: 2, value: 'RTC Recorder' }]);
const recordingType = ref<{ type: number, value: string }[]>([{ type: 1, value: 'Camera' }, { type: 2, value: 'Screening' }]);
const selectedRecorderType = ref<{ type: number, value: string }>(recorderType.value[0]);
const selectedRecordingType = ref<{ type: number, value: string }>(recordingType.value[0]);
const timerInterval = ref<number | null>(null);
const seconds = ref(0);
const minute = ref(0);
const hour = ref(0);
const formattedTime = ref('');
const rtcRecorder = ref<RecordRTC | null>(null);

onMounted(() => {
    setCameraStream();
})

async function setCameraStream() {
    if (mediaRecorder.value && isPausing.value) {
        mediaRecorder.value.stop();
    }
    if (selectedRecordingType.value.type == 1) {
        recordingStream.value = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    } else {
        await navigator.mediaDevices.getDisplayMedia({
            video: {
                displaySurface: "monitor",
            },
            audio: false,
        }).then((stream) => {
            recordingStream.value = stream;
            isRecording.value = true;
            isPausing.value = false;
            setMediaRecoder();
        }).catch((error) => {
            selectedRecordingType.value = recordingType.value[0];
            setCameraStream();
        });
    }
    displayStream();
    if (isRecording.value) {
        isRecording.value = true;
        isPausing.value = false;
        setMediaRecoder();
    }
}

function displayStream() {
    if (live.value) {
        live.value.srcObject = recordingStream.value;
        live.value.playsInline = true;
        live.value.muted = true;
        live.value.play();
    }
}

function setMediaRecoder() {
    if (recordingStream.value) {
        if (selectedRecorderType.value.type == 1) {
            mediaRecorder.value = new MediaRecorder(recordingStream.value, { mimeType: 'video/webm;codecs=h264' });
            mediaRecorder.value.ondataavailable = (event: BlobEvent) => {
                if (event.data.size > 0) {
                    totalChunks.value.push(event.data);
                }
            }
            mediaRecorder.value.addEventListener('error', () => {
                console.log('Error');
            })
            mediaRecorder.value.addEventListener('pause', () => {
                stopTimer();
            })
            mediaRecorder.value.addEventListener('resume', () => {
                startTimer();
            })
            mediaRecorder.value.addEventListener('stop', () => {
                stopRecording();
            }, false)
            mediaRecorder.value.addEventListener('start', () => {
                startRecording();
            });
            mediaRecorder.value.start();
        } else {
            rtcRecorder.value = new RecordRTC(recordingStream.value, {
                type: 'video',
                mimeType: 'video/webm',
                disableLogs: true,
                timeSlice: 1000,
                ondataavailable: function (event: any) {
                    if (event.size > 0) {
                        totalChunks.value.push(event);
                    }
                },
                checkForInactiveTracks: false,
                onTimeStamp: function (timestamp: any) {
                    console.log(timestamp);
                },
                bitsPerSecond: 128000,
                audioBitsPerSecond: 128000,
                frameInterval: 90,
                sampleRate: 96000,
                desiredSampRate: 16000,
                bufferSize: 1638,
                frameRate: 30,
                bitrate: 128000,
            });
            startTimer();
            rtcRecorder.value.startRecording();
        }
    }
}

async function stopRecording() {
    recordingStream.value?.getTracks().forEach((track) => track.stop());
    if (isCompletedStatus.value) {
        const blob = new Blob(totalChunks.value, { type: mediaRecorder.value?.mimeType });
        finalBlob.value = blob;
        console.log('Final blob:', finalBlob.value);
        preview.value!.src = URL.createObjectURL(finalBlob.value);
        isRecording.value = false;
        isPausing.value = false;
    }
    stopTimer();
}

function stopTimer() {
    if (timerInterval.value) {
        clearInterval(timerInterval.value);
        timerInterval.value = null;
    }
}

function startTimer() {
    if (!timerInterval.value)
        timerInterval.value = setInterval(() => {
            seconds.value++;
            if (seconds.value === 60) {
                seconds.value = 0;
                minute.value++;
                if (minute.value == 60) {
                    minute.value = 0;
                    hour.value++;
                }
            }
            formatTIme(hour.value, minute.value, seconds.value);
        }, 1000);
}

function formatTIme(hr: number, min: number, sec: number) {
    formattedTime.value = padZero(hr) + ':' + padZero(min) + ':' + padZero(sec);
}

function padZero(value: number) {
    return value < 10 ? '0' + value : value.toString();
}

function startRecording() {
    startTimer();
}

function startRecorder() {
    isRecording.value = true;
    isPausing.value = false;
    if (recordingStream.value)
        setMediaRecoder()
}

function pauseResumeRecorder() {
    if (selectedRecorderType.value.type == 1) {
        if (mediaRecorder.value) {
            isPausing.value = !isPausing.value;
            if (mediaRecorder.value) {
                if (mediaRecorder.value.state === 'paused')
                    mediaRecorder.value.resume();
                else
                    mediaRecorder.value.pause();
            }
        } else {
            alert('Recorder has not started yet.');
        }
    } else {
        alert('You cannot pause RTC reacorder. Pause means stop.');
    }

}

function stopRecorder() {
    if (selectedRecorderType.value.type == 1) {
        if (mediaRecorder.value) {
            isCompletedStatus.value = true;
            mediaRecorder.value.stop();
        }
    } else {
        if (rtcRecorder.value) {
            rtcRecorder.value.stopRecording(function () {
                const blob = new Blob(totalChunks.value, { type: mediaRecorder.value?.mimeType });
                finalBlob.value = blob;
                preview.value!.src = URL.createObjectURL(finalBlob.value);
                stopTimer();
            });
        }
    }
}

async function changeMode(event: Event) {
    const selectedValue = (event.target as HTMLSelectElement).value;
    const selectedIndex = recordingType.value.findIndex((item) => item.type.toString() == selectedValue);
    selectedRecordingType.value = recordingType.value[selectedIndex];
    await setCameraStream();
}

async function changeTypeMode(event: Event) {
    const selectedValue = (event.target as HTMLSelectElement).value;
    const selectedIndex = recorderType.value.findIndex((item) => item.type.toString() === selectedValue);
    selectedRecorderType.value = recorderType.value[selectedIndex];
}

function downloadVideo() {
    if (finalBlob.value) {
        const blobUrl = URL.createObjectURL(finalBlob.value);
        const downloadLink = document.createElement('a');
        downloadLink.href = blobUrl;
        downloadLink.download = 'checkvideo.mp4';
        document.body.appendChild(downloadLink);
        downloadLink.click();
        document.body.removeChild(downloadLink)
    }
}

function resetSetup() {
    recordingStream.value = null;
    totalChunks.value = [];
    isRecording.value = false;
    isPausing.value = false;
    finalBlob.value = null;
    isCompletedStatus.value = false;
    timerInterval.value = null;
    seconds.value = 0;
    minute.value = 0;
    hour.value = 0;
    formattedTime.value = '';
    mediaRecorder.value = null;
    rtcRecorder.value = null;
    selectedRecorderType.value = recorderType.value[0];
    selectedRecordingType.value = recordingType.value[0];
    setCameraStream();
}
</script>
<template>
    <div class="header">Different mode recording test.</div>
    <div class="parent-div">
        <div class="video-container">
            <div class="recorder-div">
                <div class="recorder-type-container" v-if="!isRecording">
                    <select name="recorder-type" id="recorder-type" class="selection-mode" @change="changeTypeMode($event)"
                        :value="selectedRecorderType.type">
                        <option v-for="(rtype, index) in recorderType" :value="rtype.type" :key="index">{{ rtype.value }}
                        </option>
                    </select>
                </div>
                <div class="recording-timer" v-if="formattedTime">
                    {{ formattedTime }}
                </div>
                <video id="live" ref="live" autoplay muted></video>
                <div class="control-button-container">
                    <div>
                        <div class="selection-mode-container">
                            <select class="selection-mode" name="selection-mode" id="selection-mode"
                                :value="selectedRecordingType.type" @change="changeMode($event)"
                                v-if="!isRecording || (isRecording && isPausing)">
                                <option v-for="(stype, jndex) in recordingType" :key="jndex" :value="stype.type">{{
                                    stype.value
                                }}
                                </option>
                            </select>
                        </div>
                        <div class="record-button" @click="startRecorder" v-if="!isRecording"></div>
                        <div class="stop-button" @click="stopRecorder" v-else>
                            <div></div>
                        </div>
                        <div class="pause-buton" @click="pauseResumeRecorder">
                            <div v-if="!isPausing">
                                ||
                            </div>
                            <div class="resume" v-else></div>
                        </div>
                    </div>
                </div>
            </div>
            <div class="preview-div">
                <video id="preview" ref="preview" autoplay controls></video>
                <div class="download-button-container">
                    <button @click="downloadVideo">Download</button>
                </div>
            </div>
        </div>
    </div>
    <div class="reset-button-container">
        <button @click="resetSetup">Reset</button>
    </div>
</template>
<style>
.header {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 40px;
    font-weight: bold;
    text-overflow: ellipsis;
    overflow: hidden;
    white-space: nowrap;
}

.recorder-type-container {
    display: flex;
    justify-content: center;
    align-items: center;
    position: absolute;
    width: 100%;
    top: 10px;
    z-index: 1;
}

.parent-div {
    height: 100%;
}

.video-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    height: calc(100vh - 100px);
}

.video-container>div {
    width: 100%;
    margin: 5px;
    border: 1px solid black;
    position: relative;
    height: 100%;
}

.video-container>div>video {
    max-width: 100%;
    max-height: 100%;
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);
}

.control-button-container {
    position: absolute;
    bottom: 5px;
    width: 100%;
}

.control-button-container>div {
    display: flex;
    justify-content: center;
    align-items: center;
}

.record-button {
    height: 35px;
    width: 35px;
    background: red;
    border-radius: 50%;
    border: 5px solid black;
    box-shadow: 0px 0px 2px 2px white;
    cursor: pointer;
}

.stop-button {
    height: 40px;
    width: 40px;
    background: white;
    border-radius: 50%;
    border: 1px solid grey;
    box-shadow: 0px 0px 2px 2px white;
    cursor: pointer;
    position: relative;
}

.stop-button>div {
    height: 20px;
    width: 20px;
    background-color: red;
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);
}

.pause-buton {
    height: 35px;
    width: 35px;
    background: white;
    border: 1px solid grey;
    border-radius: 50%;
    margin-left: 10px;
    box-shadow: 0px 0px 2px 2px white;
    cursor: pointer;
    position: relative;
}

.pause-buton>div {
    position: absolute;
    top: 52%;
    left: 52%;
    transform: translate(-50%, -50%);
}

.selection-mode-container {
    margin-right: 5px;
    width: 100px;
}

.selection-mode {
    padding: 8px;
    border-radius: 13px;
    border: 1px solid grey;
    box-shadow: 0px 0px 2px 2px white;
    color: darkslategrey;
}

.resume {
    width: 0;
    height: 0;
    border-top: 6px solid transparent;
    border-left: 12px solid #555;
    border-bottom: 8px solid transparent;
}

.recording-timer {
    position: absolute;
    color: red;
    left: 50%;
    transform: translateX(-50%);
    bottom: 55px;
    z-index: 1;
    text-shadow: 0px -2px 2px white;
    background: white;
    padding: 5px;
    opacity: 0.5;
}

.reset-button-container {
    display: flex;
    justify-content: center;
    align-items: center;
}

.reset-button-container>button {
    border: 1px solid darkgrey;
    padding: 5px 10px;
    color: red;
    margin-top: 10px;
    cursor: pointer;
}

.download-button-container {
    position: absolute;
    display: flex;
    justify-content: center;
    width: 100%;
    align-items: center;
    bottom: 10px;
}

.download-button-container>button {
    border: 1px solid darkgrey;
    padding: 5px 10px;
    color: red;
    margin-top: 10px;
    cursor: pointer;
}
</style>