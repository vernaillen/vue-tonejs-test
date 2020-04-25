<template>
    <div v-bind:class="keyClass"
         @mousedown="$emit('hit')"
         @mouseup="$emit('release')"
         @mouseout="$emit('release')">
        <p>{{ tone }}</p>
    </div>
</template>

<script>
    export default {
        name: 'piano-key',
        props: {
            tone: {
                type: String,
                required: true
            },
            isKeySelected: {
                type: Boolean,
                required: true
            }
        },
        computed: {
            keyClass: function () {
                let keyClass = "key"
                if (this.tone.includes('#')) keyClass += " major"
                if (this.isKeySelected) keyClass += " selected"
                return keyClass
            }
        }
    }
</script>

<style scoped>
    .key {
        border: 1px solid #CCCCCC;
        float: left;
        height: 250px;
        width: 80px;
        position: relative;
        z-index: 1;
    }

    .key p {
        bottom: 0;
        color: black;
        cursor: default;
        font-size: 11px;
        text-align: center;
        position: absolute;
        width: 100%;

        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }

    .key.major {
        background: #000000;
        border: none;
        height: 180px;
        width: 50px;
        margin-left: -27px;
        position: relative;
        z-index: 2;
    }

    .key.major p {
        color: #FFFFFF;
    }

    .key.major + .key {
        margin-left: -25px;
    }

    .key.selected {
        background: #DDDDDD;
    }
    .key.major.selected {
        background: #444444;
    }
</style>
