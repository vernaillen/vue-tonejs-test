<template>
    <knob-control
            class="knob-control"
            :min="0"
            :max="127"
            primary-color="#E844C3"
            secondary-color="#E7B6DC"
            text-color="#E844C3"
            v-model="input">
    </knob-control>
</template>

<script>
    import KnobControl from 'vue-knob-control'

    export default {
        name: "Knob",
        components: {
            KnobControl
        },
        props: {
            input: {
                required: true
            }
        },
        watch: {
            input: function (val) {
                this.$emit('changed', val)
            }
        }
    }
</script>

<style scoped>
    .knob-control {
        display: inline-block;
        margin: 0 10px;
    }
</style>
