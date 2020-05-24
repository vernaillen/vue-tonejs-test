<template>
    <v-container class=" py-0 px-2">
        <div  id="piano">
            <piano-key v-for="tone in tones"
                       :tone="tone"
                       :isKeySelected="isKeySelected(tone)"
                       :key="tone"
                        @hit="soundOn(tone, keyVelocity)"
                        @release="soundOff">
            </piano-key>
        </div>
        <p class="controls">
            oscillator waveform:
            <v-radio-group v-model="selectedWaveform" row>
                <template v-for="waveform in waveforms">
                    <v-radio :label="waveform" :value="waveform" :key="waveform"></v-radio>
                </template>
            </v-radio-group>
        </p>
        <p class="knobs">
            velocity:<br/><br/>
            <knob :input="keyVelocity" @changed="updateKeyVelocity"></knob>
        </p>
        <p class="metadata">
            key: {{ selectedKey }}<br />
            midi: {{ midiMessage[0] }}, {{ midiMessage[1] }}, {{ midiMessage[2] }}<br/>
        </p>
        <p>
            <FFT :analyser="fftAnalyser"></FFT>
            <waveform :analyser="waveformAnalyser"></waveform>
        </p>
        <p>
            <v-icon @click="startAudioPlayer">fas fa fa-play</v-icon>
            <v-icon @click="stopAudioPlayer">fas fa-stop</v-icon>
            <br/><br/><br/>
        </p>
        <p>
            <audio-motion :audio-node="audioPlayer" :audio-context="audioContext"/>
        </p>
    </v-container>
</template>

<script>
    import PianoKey from './PianoKey.vue'
    import Waveform from './Waveform'
    import FFT from './FFT'
    import Knob from './Knob'
    import Tone from 'tone'
    import AudioMotion from "./AudioMotion";
    Tone.context.latencyHint = "fastest";

    export default {
        name: 'piano',
        components: {
            AudioMotion,
            PianoKey,
            Waveform,
            FFT,
            Knob
        },
        props: {
            audioContext: {
                required: true
            }
        },
        data () {
            return {
                keyToPitch: { "a":"C3", "z":"C#3", "e":"D3", "r":"D#3", "t":"E3", "y":"F3", "u":"F#3", "i":"G3", "o":"G#3", "p":"A3", "^":"A#3", "$":"B3" },
                tones: ['C3', 'C#3', 'D3', 'D#3', 'E3', 'F3', 'F#3', 'G3', 'G#3', 'A3', 'A#3', 'B3', 'C4', 'C#4', 'D4', 'D#4', 'E4', 'F4', 'F#4', 'G4', 'G#4', 'A4', 'A#4', 'B4'],
                waveforms: [ 'sine', 'square', 'triangle', 'sawtooth'],
                selectedKey: '',
                selectedWaveform: 'sine',
                midiMessage: [0, 0, 0],
                keyVelocity: 100
            }
        },
        computed: {
            audioPlayer: function () {
                return new Tone.Player('https://api.soundcloud.com/tracks/672555005/stream?client_id=1745017edcfeb72a175c95614a1cc212')
                    .fan(this.fftAnalyser, this.waveformAnalyser)
                    .toMaster();
            },
            synth: function () {
                return new Tone.Synth({
                    oscillator : {
                        type : this.selectedWaveform,
                        frequency: this.oscFrequency
                    }
                }).fan(this.fftAnalyser, this.waveformAnalyser).toMaster();
            },
            fftAnalyser: function () {
                return new Tone.Analyser({
                    size: 16384,
                    smoothing: 0.8,
                    type: 'fft',
                    maxDecibels: 10,
                    minDecibels: -100
                }).toMaster()
            },
            waveformAnalyser: function () {
                return new Tone.Analyser({
                    type: 'waveform',
                    size: 1024,
                    smoothing: 0.1,
                    maxDecibels: 10,
                    minDecibels: -100
                }).toMaster()
            }
        },
        created() {
            window.addEventListener('keydown', this.onkeydown);
            window.addEventListener('keyup', this.onkeyup);
            new Promise((resolve, reject) => {
                if (navigator.requestMIDIAccess)
                    navigator.requestMIDIAccess()
                        .then(resolve)
                        .catch(reject);
                else reject();
            }).then(access => {
                const devices = access.inputs.values();
                for (let device of devices) {
                    device.onmidimessage = this.onMessage.bind(device);
                }
            })
            this.audioPlayer.autostart = false
        },
        methods: {
            isKeySelected: function (tone) {
                return this.selectedKey == tone
            },
            soundOn: function (tone, velocity) {
                this.synth.triggerAttack(tone, 0, velocity / 127)
                this.selectedKey = tone
            },
            soundOff: function () {
                this.synth.triggerRelease();
                this.selectedKey = ''
            },
            onMessage(message) {
                let [action, midiKey, velocity] = message.data;
                if (action == 144) {
                    this.midiMessage = message.data;
                    let index = midiKey - 60;
                    if (index >= 0 && index < this.tones.length) {
                        this.soundOn(this.tones[index], velocity)
                        this.keyVelocity = velocity
                    }
                } else if (action == 153) {
                    this.midiMessage = message.data;
                    let index = midiKey - 60;
                    if (index >= 0 && index < this.tones.length) {
                        this.soundOn(this.tones[index], velocity)
                    }
                } else if (action == 176 && midiKey == 80) {
                    this.midiMessage = message.data;
                    this.oscFrequency = velocity
                } else if (action == 176 && midiKey == 82) {
                    this.midiMessage = message.data;
                    this.keyVelocity = velocity
                } else if (action == 128 || action == 137 || action == 217) {
                    this.soundOff()
                } else if (action == 248) {
                    // do nothing
                } else {
                    this.midiMessage = message.data;
                }
            },
            onkeydown(e) {
                this.soundOn(this.keyToPitch[e.key], this.keyVelocity)
            },
            onkeyup() {
                this.soundOff()
            },
            updateKeyVelocity(value) {
                this.keyVelocity = value
            },
            startAudioPlayer() {
                this.audioPlayer.volume.value= 4
                this.audioPlayer.start()
            },
            stopAudioPlayer() {
                this.audioPlayer.stop()
            }
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    #piano {
        display: inline-block;
        margin: 50px auto;
    }
    p.metadata, .knobs {
        display: block;
        margin: 10px auto;
    }
</style>
