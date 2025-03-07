<template>
<q-page class="send">
    <template v-if="view_only">
        <div class="q-pa-md">
            {{ $t("strings.viewOnlyMode") }}
        </div>
    </template>
    <template v-else>
        <div class="q-pa-md">
            <div class="row gutter-md">

                <!-- Amount -->
                <div class="col-6">
                    <OscillateField :label="$t('fieldLabels.amount')" :error="$v.newTx.amount.$error">
                        <q-input v-model="newTx.amount"
                            :dark="theme=='dark'"
                            type="number"
                            min="0"
                            :max="unlocked_balance / 1e9"
                            placeholder="0"
                            @blur="$v.newTx.amount.$touch"
                            hide-underline
                        />
                        <q-btn color="secondary" @click="newTx.amount = unlocked_balance / 1e9" :text-color="theme=='dark'?'white':'dark'">
                            {{ $t("buttons.all") }}
                        </q-btn>
                    </OscillateField>
                </div>

                <!-- Priority -->
                <div class="col-6">
                    <OscillateField :label="$t('fieldLabels.priority')">
                        <q-select :dark="theme=='dark'"
                            v-model="newTx.priority"
                            :options="priorityOptions"
                            hide-underline
                        />
                    </OscillateField>
                </div>
            </div>

            <!-- Address -->
            <div class="col q-mt-sm">
                <OscillateField :label="$t('fieldLabels.address')" :error="$v.newTx.address.$error">
                     <q-input v-model="newTx.address"
                        :dark="theme=='dark'"
                        @blur="$v.newTx.address.$touch"
                        :placeholder="address_placeholder"
                        hide-underline
                    />
                    <q-btn color="secondary" :text-color="theme=='dark'?'white':'dark'" to="addressbook">
                        {{ $t("buttons.contacts") }}
                    </q-btn>
                </OscillateField>
            </div>

            <!-- Payment ID -->
            <div class="col q-mt-sm">
                <OscillateField :label="$t('fieldLabels.paymentId')" :error="$v.newTx.payment_id.$error" optional>
                     <q-input v-model="newTx.payment_id"
                        :dark="theme=='dark'"
                        @blur="$v.newTx.payment_id.$touch"
                        :placeholder="$t('placeholders.hexCharacters', { count: '16 or 64' })"
                        hide-underline
                    />
                </OscillateField>
            </div>

            <!-- Notes -->
            <div class="col q-mt-sm">
                <OscillateField :label="$t('fieldLabels.notes')" optional>
                     <q-input v-model="newTx.note"
                        type="textarea"
                        :dark="theme=='dark'"
                        :placeholder="$t('placeholders.transactionNotes')"
                        hide-underline
                    />
                </OscillateField>
            </div>

            <!-- Save to address book -->
            <q-field>
                <q-checkbox v-model="newTx.address_book.save" :label="$t('strings.saveToAddressBook')" :dark="theme=='dark'" />
            </q-field>

            <div v-if="newTx.address_book.save">
                <OscillateField :label="$t('fieldLabels.name')" optional>
                     <q-input v-model="newTx.address_book.name"
                        :dark="theme=='dark'"
                        :placeholder="$t('placeholders.addressBookName')"
                        hide-underline
                    />
                </OscillateField>
                <OscillateField class="q-mt-sm" :label="$t('fieldLabels.notes')" optional>
                     <q-input v-model="newTx.address_book.description"
                        type="textarea"
                        rows="2"
                        :dark="theme=='dark'"
                        :placeholder="$t('placeholders.additionalNotes')"
                        hide-underline
                    />
                </OscillateField>
            </div>

            <q-field class="q-pt-sm">
                <q-btn
                    class="send-btn"
                    :disable="!is_able_to_send"
                    color="primary" @click="send()" :label="$t('buttons.send')" />
            </q-field>

        </div>

        <q-inner-loading :visible="tx_status.sending" :dark="theme=='dark'">
            <q-spinner color="blue" :size="30" />
        </q-inner-loading>

    </template>

</q-page>
</template>

<script>
import { mapState } from "vuex"
import { required, decimal } from "vuelidate/lib/validators"
import { payment_id, address, greater_than_zero } from "src/validators/common"
import Identicon from "components/identicon"
import OscillateField from "components/oscillate_field"
import WalletPassword from "src/mixins/wallet_password"
const objectAssignDeep = require("object-assign-deep");

export default {
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        view_only: state => state.gateway.wallet.info.view_only,
        unlocked_balance: state => state.gateway.wallet.info.unlocked_balance,
        tx_status: state => state.gateway.tx_status,
        is_ready (state) {
            return this.$store.getters["gateway/isReady"]
        },
        is_able_to_send (state) {
            return this.$store.getters["gateway/isAbleToSend"]
        },
        address_placeholder (state) {
            const wallet = state.gateway.wallet.info;
            const prefix = (wallet && wallet.address && wallet.address[0]) || "a";
            return `${prefix}..`;
        }
    }),
    data () {
        return {
            sending: false,
            newTx: {
                amount: 0,
                address: "",
                payment_id: "",
                priority: 0,
                address_book: {
                    save: false,
                    name: "",
                    description: ""
                }
            },
            priorityOptions: [
                {label: this.$t("strings.priorityOptions.automatic"), value: 0},
                {label: this.$t("strings.priorityOptions.slow"), value: 1},
                {label: this.$t("strings.priorityOptions.normal"), value: 2},
                {label: this.$t("strings.priorityOptions.fast"), value: 3},
                {label: this.$t("strings.priorityOptions.fastest"), value: 4},
            ],
        }
    },
    validations: {
        newTx: {
            amount: {
                required,
                decimal,
                greater_than_zero
            },
            address: {
            required,
            isAddress(value) {
                    if (value === '') return true

                    return new Promise(resolve => {
                        address(value, this.$gateway)
                            .then(() => resolve(true))
                            .catch(e => resolve(false))
                    });
                }
            },
            payment_id: { payment_id }
        }
    },
    watch: {
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
                        this.$v.$reset();
                        this.newTx = {
                            amount: 0,
                            address: "",
                            payment_id: "",
                            priority: 0,
                            address_book: {
                                save: false,
                                name: "",
                                description: ""
                            },
                            note: ""
                        }
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 1000,
                            message: this.tx_status.message
                        })
                        break;
                }
            },
            deep: true
        },
        $route (to) {
            if(to.path == "/wallet/send" && to.query.hasOwnProperty("address")) {
                this.autoFill(to.query)
            }
        }
    },
    mounted () {
        if(this.$route.path == "/wallet/send" && this.$route.query.hasOwnProperty("address")) {
            this.autoFill(this.$route.query)
        }
    },
    methods: {

        autoFill: function (info) {
            this.newTx.address = info.address
            this.newTx.payment_id = info.payment_id
        },

        send: function () {
            this.$v.newTx.$touch()

            if(this.newTx.amount < 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.negativeAmount")
                })
                return
            } else if(this.newTx.amount == 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.zeroAmount")
                })
                return
            } else if(this.newTx.amount > this.unlocked_balance / 1e9) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.notEnoughBalance")
                })
                return
            } else if (this.$v.newTx.amount.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidAmount")
                })
                return
            }


            if (this.$v.newTx.address.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidAddress")
                })
                return
            }

            if (this.$v.newTx.payment_id.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidPaymentId")
                })
                return
            }

            this.showPasswordConfirmation({
                title: this.$t("dialog.transfer.title"),
                noPasswordMessage: this.$t("dialog.transfer.message"),
                ok: {
                    label: this.$t("dialog.transfer.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_tx_status", {
                    code: 1,
                    message: "Sending transaction",
                    sending: true
                })
                const newTx = objectAssignDeep.noMutate(this.newTx, {password})
                this.$gateway.send("wallet", "transfer", newTx)
            }).catch(() => {
            })
        }
    },
    mixins: [WalletPassword],
    components: {
        Identicon,
        OscillateField
    }
}
</script>

<style lang="scss">
.send {
    .send-btn {
        width: 200px;
    }
}
</style>
