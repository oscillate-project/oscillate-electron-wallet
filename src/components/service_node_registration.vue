<template>
<div class="service-node-registration">
    <div class="q-pa-md">
        <i18n path="strings.serviceNodeRegistrationDescription" tag="div" class="description q-mb-lg">
            <b place="registerCommand">register_service_node</b>
            <b place="prepareCommand">prepare_registration</b>
        </i18n>
        <OscillateField :label="$t('fieldLabels.serviceNodeCommand')" :error="$v.registration_string.$error" :disabled="registration_status.sending">
            <q-input
                v-model="registration_string"
                type="textarea"
                :dark="theme=='dark'"
                @blur="$v.registration_string.$touch"
                placeholder="register_service_node ..."
                :disabled="registration_status.sending"
                hide-underline
            />
        </OscillateField>
        <q-field class="q-pt-sm">
            <q-btn color="primary" @click="register()" :label="$t('buttons.registerServiceNode')" :disabled="registration_status.sending"/>
        </q-field>
    </div>

    <q-inner-loading :visible="registration_status.sending" :dark="theme=='dark'">
        <q-spinner color="blue" :size="30" />
    </q-inner-loading>
</div>
</template>

<script>
const objectAssignDeep = require("object-assign-deep");
import { mapState } from "vuex"
import { required } from "vuelidate/lib/validators"
import OscillateField from "components/oscillate_field"
import WalletPassword from "src/mixins/wallet_password"

export default {
    name: "ServiceNodeRegistration",
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        registration_status: state => state.gateway.service_node_status.registration,
    }),
    data () {
        return {
            registration_string: "",
        }
    },
    validations: {
        registration_string: { required }
    },
    watch: {
        registration_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.registration_status.code) {
                    case 0:
                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.registration_status.message
                        })
                        this.$v.$reset();
                        this.registration_string = ""
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.registration_status.message
                        })
                        break;
                }
            },
            deep: true
        },
    },
    methods: {
        register: function() {
            this.$v.registration_string.$touch()

            if (this.$v.registration_string.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidServiceNodeCommand")
                })
                return
            }

            this.showPasswordConfirmation({
                title: this.$t("dialog.registerServiceNode.title"),
                noPasswordMessage: this.$t("dialog.registerServiceNode.message"),
                ok: {
                    label: this.$t("dialog.registerServiceNode.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_snode_status", {
                    registration: {
                        code: 1,
                        message: "Registering...",
                        sending: true
                    }
                })
                this.$gateway.send("wallet", "register_service_node", {
                    password,
                    string: this.registration_string
                })
            }).catch(() => {
            })

        }
    },
    mixins: [WalletPassword],
    components: {
        OscillateField
    }
}
</script>

<style lang="scss">
</style>
