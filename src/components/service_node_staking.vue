<template>
<div class="service-node-staking">
    <div class="q-px-md q-pt-md">
        <OscillateField :label="$t('fieldLabels.serviceNodeKey')" :error="$v.service_node.key.$error">
            <q-input v-model="service_node.key"
                :dark="theme=='dark'"
                @blur="$v.service_node.key.$touch"
                :placeholder="$t('placeholders.hexCharacters', { count: 64 })"
                hide-underline
            />
        </OscillateField>

        <OscillateField :label="$t('fieldLabels.amount')" class="q-mt-md" :error="$v.service_node.amount.$error">
            <q-input v-model="service_node.amount"
                :dark="theme=='dark'"
                type="number"
                min="0"
                :max="unlocked_balance / 1e9"
                placeholder="0"
                @blur="$v.service_node.amount.$touch"
                hide-underline
            />
            <q-btn color="secondary" @click="service_node.amount = unlocked_balance / 1e9" :text-color="theme=='dark'?'white':'dark'">
                {{ $t("buttons.all") }}
            </q-btn>
        </OscillateField>



        <q-field class="buttons q-pt-sm">
            <q-btn
                :disable="!is_able_to_send"
                color="primary" @click="stake()" :label="$t('buttons.stake')" />
            <q-btn
                :disable="!is_able_to_send"
                color="secondary" @click="sweepAllWarning()" :label="$t('buttons.sweepAll')" />
        </q-field>

    </div>

    <ServiceNodeUnlock />

    <q-inner-loading :visible="stake_status.sending || tx_status.sending" :dark="theme=='dark'">
        <q-spinner color="blue" :size="30" />
    </q-inner-loading>
</div>
</template>

<script>
const { clipboard } = require("electron")
const objectAssignDeep = require("object-assign-deep");
import { mapState } from "vuex"
import { required, decimal } from "vuelidate/lib/validators"
import { i18n } from "plugins/i18n"
import { payment_id, service_node_key, greater_than_zero, address } from "src/validators/common"
import OscillateField from "components/oscillate_field"
import WalletPassword from "src/mixins/wallet_password"
import ServiceNodeUnlock from "components/service_node_unlock"

export default {
    name: "ServiceNodeStaking",
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        unlocked_balance: state => state.gateway.wallet.info.unlocked_balance,
        info: state => state.gateway.wallet.info,
        stake_status: state => state.gateway.service_node_status.stake,
        tx_status: state => state.gateway.tx_status,
        award_address: state => state.gateway.wallet.info.address,
        is_ready (state) {
            return this.$store.getters["gateway/isReady"]
        },
        is_able_to_send (state) {
            return this.$store.getters["gateway/isAbleToSend"]
        },
        address_placeholder (state) {
            const wallet = state.gateway.wallet.info;
            const prefix = (wallet && wallet.address && wallet.address[0]) || "L";
            return `${prefix}..`;
        },
    }),
    data () {
        return {
            service_node: {
                key: "",
                amount: 0,
            },
        }
    },
    validations: {
        service_node: {
            key: { required, service_node_key },
            amount: {
                required,
                decimal,
                greater_than_zero,
            },
        }
    },
    watch: {
        stake_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.stake_status.code) {
                    case 0:
                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.stake_status.message
                        })
                        this.$v.$reset();
                        this.service_node = {
                            key: "",
                            amount: 0,
                        }
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.stake_status.message
                        })
                        break;
                }
            },
            deep: true
        },
        tx_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.tx_status.code) {
                    case 0:
                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.tx_status.message
                        })
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.tx_status.message
                        })
                        break;
                }
            },
            deep: true
        },
    },
    methods: {
        sweepAllWarning: function () {
            this.$q.dialog({
                title: this.$t("dialog.sweepAllWarning.title"),
                message: this.$t("dialog.sweepAllWarning.message"),
                ok: {
                    label: this.$t("dialog.sweepAllWarning.ok")
                },
                cancel: {
                    flat: true,
                    label: this.$t("dialog.buttons.cancel"),
                    color: this.theme === "dark" ? "white" : "dark"
                }
            }).then(() => {
                this.sweepAll()
            }).catch(() => {})
        },
        sweepAll: function () {
            const { unlocked_balance } = this.info;

            const tx = {
                amount: unlocked_balance / 1e9,
                address: this.award_address,
                priority: 0
            };

            this.showPasswordConfirmation({
                title: this.$t("dialog.sweepAll.title"),
                noPasswordMessage: this.$t("dialog.sweepAll.message"),
                ok: {
                    label: this.$t("dialog.sweepAll.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_tx_status", {
                    code: 1,
                    message: "Sweeping all",
                    sending: true
                })
                const newTx = objectAssignDeep.noMutate(tx, {password})
                this.$gateway.send("wallet", "transfer", newTx)
            }).catch(() => {
            })
        },
        stake: function () {
            this.$v.service_node.$touch()

            if (this.$v.service_node.key.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidServiceNodeKey")
                })
                return
            }

            if(this.service_node.amount < 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.negativeAmount")
                })
                return
            } else if(this.service_node.amount == 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.zeroAmount")
                })
                return
            } else if(this.service_node.amount > this.unlocked_balance / 1e9) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.notEnoughBalance")
                })
                return
            } else if (this.$v.service_node.amount.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidAmount")
                })
                return
            }

            this.showPasswordConfirmation({
                title: this.$t("dialog.stake.title"),
                noPasswordMessage: this.$t("dialog.stake.message"),
                ok: {
                    label: this.$t("dialog.stake.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_snode_status", {
                    stake: {
                        code: 1,
                        message: "Staking...",
                        sending: true
                    }
                })
                const service_node = objectAssignDeep.noMutate(this.service_node, {
                    password,
                    destination: this.award_address
                })

                this.$gateway.send("wallet", "stake", service_node)
            }).catch(() => {
            })
        }
    },
    mixins: [WalletPassword],
    components: {
        OscillateField,
        ServiceNodeUnlock
    }
}
</script>

<style lang="scss">
.service-node-staking {
    .buttons {
        .q-btn:not(:first-child) {
            margin-left: 8px;
        }
    }
}
</style>
