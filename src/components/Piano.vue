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
            oscillator type: {{ selectedWaveform }}
        </p>
        <p>
            <br>
            vue-p5 test (currently only showing velocity):
            <waveform :input="keyVelocity"></waveform>
        </p>
    </v-container>
</template>

<script>
    import PianoKey from './PianoKey.vue'
    import Waveform from './Waveform'
    import Knob from './Knob'
    import Tone from 'tone'
    Tone.context.latencyHint = "fastest";

    export default {
        name: 'piano',
        components: {
            PianoKey,
            Waveform,
            Knob
        },
        data () {
            return {
                keyToPitch: { "a":"C3", "z":"C#3", "e":"D3", "r":"D#3", "t":"E3", "y":"F3", "u":"F#3", "i":"G3", "o":"G#3", "p":"A3", "^":"A#3", "$":"B3" },
                tones: ['C3', 'C#3', 'D3', 'D#3', 'E3', 'F3', 'F#3', 'G3', 'G#3', 'A3', 'A#3', 'B3', 'C4', 'C#4', 'D4', 'D#4', 'E4', 'F4', 'F#4', 'G4', 'G#4', 'A4', 'A#4', 'B4'],
                waveforms: [ 'sine', 'square', 'triangle', 'sawtooth'],
                selectedKey: '',
                selectedWaveform: 'square',
                midiMessage: [0, 0, 0],
                oscFrequency: 0,
                keyVelocity: 100
            }
        },
        computed: {
            synth: function () {
                return new Tone.Synth({
                    oscillator : {
                        type : this.selectedWaveform,
                        frequency: this.oscFrequency
                    }
                }).toMaster();
            },
            frequency: function () {
                return this.synth.oscillator.frequency.value
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

        },
        methods: {
            isKeySelected: function (tone) {
                return this.selectedKey == tone
            },
            soundOn: function (tone, velocity) {
                console.log('tone: ' + tone)
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
