<template>
<div class="oscillate-field" :class="{disable, 'disable-hover': disableHover}">
    <div class="label row items-center" v-if="label" :disabled="disable">
        {{ label }}
        <span v-if="optional" class="optional">({{ $t("fieldLabels.optional") }})</span>
    </div>
    <div class="content row items-center" :class="{error}">
        <slot></slot>
    </div>
    <div class="error-label" v-if="error && errorLabel" :disabled="disable">{{ errorLabel }}</div>
</div>
</template>

<script>
export default {
    name: "OscillateField",
    props: {
        label: {
            type: String,
            required: false
        },
        error: {
            type: Boolean,
            required: false,
            default: false
        },
        errorLabel: {
            type: String,
            required: false
        },
        optional: {
            type: Boolean,
            required: false,
            default: false
        },
        disable: {
            type: Boolean,
            required: false,
            default: false
        },
        disableHover: {
            type: Boolean,
            required: false,
            default: false
        }
    },
    data () {
        return {
        }
    },
}
</script>

<style lang="scss">
.oscillate-field {
    .label {
        margin: 6px 0;
        font-weight: bold;
        font-size: 12px;

        // Disable text selection
        -webkit-user-select: none;
        user-select: none;
        cursor: default;

        .optional {
            font-weight: 400;
            margin-left: 4px;
        }
    }
    .content {
        border-radius: 3px;
        padding: 6px 8px;
        min-height: 46px;

        > * {
            margin-right: 4px;
        }

        > *:last-child {
            margin-right: 0px;
        }

        .q-input, .q-select, .q-datetime-input {
            flex: 1;
        }

        .q-btn {
            padding: 4px 8px;
            font-size: 12px !important;
        }
    }
}
</style>
