<template>
<q-page>
    <div class="q-mx-md">
        <OscillateField class="q-mt-md" :label="$t('fieldLabels.walletName')" :error="$v.wallet.name.$error">
            <q-input
                v-model="wallet.name"
                :placeholder="$t('placeholders.walletName')"
                @keyup.enter="restore_wallet"
                @blur="$v.wallet.name.$touch"
                :dark="theme=='dark'"
                hide-underline
                />
        </OscillateField>

        <OscillateField class="q-mt-md" :label="$t('fieldLabels.mnemonicSeed')" :error="$v.wallet.seed.$error">
            <q-input
                v-model="wallet.seed"
                :placeholder="$t('placeholders.mnemonicSeed')"
                type="textarea"
                @blur="$v.wallet.seed.$touch"
                :dark="theme=='dark'"
                hide-underline
                />
        </OscillateField>

        <div class="row items-end q-mt-md">
            <div class="col">
                <OscillateField v-if="wallet.refresh_type=='date'" :label="$t('fieldLabels.restoreFromDate')">
                    <q-datetime v-model="wallet.refresh_start_date" type="date"
                                modal :min="1525305600000" :max="Date.now()"
                                :dark="theme=='dark'"
                                hide-underline
                                />
                </OscillateField>
                <OscillateField v-else-if="wallet.refresh_type=='height'" :label="$t('fieldLabels.restoreFromBlockHeight')" :error="$v.wallet.refresh_start_height.$error">
                    <q-input v-model="wallet.refresh_start_height" type="number"
                                min="0"
                                @blur="$v.wallet.refresh_start_height.$touch"
                                :dark="theme=='dark'"
                                hide-underline
                                />
                </OscillateField>
            </div>
            <div class="col-auto q-ml-sm">
                <template v-if="wallet.refresh_type=='date'">
                    <q-btn @click="wallet.refresh_type='height'" class="float-right" :text-color="theme=='dark'?'white':'dark'" flat>
                        <div style="min-width: 80px; height: 38px;" class="text-center flex column items-center justify-center">
                            <q-icon class="block" name="clear_all" />
                            <div style="font-size:10px">
                                {{ $t("strings.switchToHeightSelect") }}
                            </div>
                        </div>
                    </q-btn>
                </template>
                <template v-else-if="wallet.refresh_type=='height'">
                    <q-btn @click="wallet.refresh_type='date'" class="float-right" :text-color="theme=='dark'?'white':'dark'" flat>
                        <div style="min-width: 80px; height: 38px;" class="text-center flex column items-center justify-center">
                            <q-icon class="block" name="today" />
                            <div style="font-size:10px">
                                {{ $t("strings.switchToDateSelect") }}
                            </div>
                        </div>
                    </q-btn>
                </template>
            </div>
        </div>

        <OscillateField class="q-mt-md" :label="$t('fieldLabels.password')">
            <q-input
                v-model="wallet.password"
                :placeholder="$t('placeholders.walletPassword')"
                @keyup.enter="restore_wallet"
                type="password"
                :dark="theme=='dark'"
                hide-underline
                />
        </OscillateField>

        <OscillateField class="q-mt-md" :label="$t('fieldLabels.confirmPassword')">
            <q-input
                v-model="wallet.password_confirm"
                @keyup.enter="restore_wallet"
                type="password"
                :dark="theme=='dark'"
                hide-underline
                />
        </OscillateField>
        <PasswordStrength :password="wallet.password" ref="password_strength" />
        <q-field>
            <q-btn color="primary" @click="restore_wallet" :label="$t('buttons.restoreWallet')" />
        </q-field>

    </div>
</q-page>
</template>

<script>
import PasswordStrength from "components/password_strength"
import { required, numeric } from "vuelidate/lib/validators"
import { mapState } from "vuex"
import OscillateField from "components/oscillate_field"
export default {
    data () {
        return {
            wallet: {
                name: "",
                seed: "",
                refresh_type: "date",
                refresh_start_height: 0,
                refresh_start_date: 1525305600000, // timestamp of block 1
                password: "",
                password_confirm: ""
            },
        }
    },
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        status: state => state.gateway.wallet.status,
    }),
    watch: {
        status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.status.code) {
                    case 1:
                        break;
                    case 0:
                        this.$q.loading.hide()
                        this.$router.replace({ path: "/wallet-select/created" });
                        break;
                    default:
                        this.$q.loading.hide()
                        this.$q.notify({
                            type: "negative",
                            timeout: 1000,
                            message: this.status.message
                        })
                        break;
                }
            },
            deep: true
        }
    },
    validations: {
        wallet: {
            name: { required },
            seed: { required },
            refresh_start_height: { numeric }
        }
    },
    methods: {
        restore_wallet() {
            this.$v.wallet.$touch()

            if (this.$v.wallet.name.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.enterWalletName")
                })
                return
            }
            if (this.$v.wallet.seed.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.enterSeedWords")
                })
                return
            }

            let seed = this.wallet.seed.trim()
                .replace(/\n/g, " ")
                .replace(/\t/g, " ")
                .replace(/\s{2,}/g, " ")
                .split(" ")
            if(seed.length !== 14 && seed.length !== 24 && seed.length !== 25 && seed.length !== 26) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidSeedLength")
                })
                return
            }

            if (this.$v.wallet.refresh_start_height.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidRestoreHeight")
                })
                return
            }
            if(this.wallet.password != this.wallet.password_confirm) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.passwordNoMatch")
                })
                return
            }

            this.$q.loading.show({
                delay: 0
            })

            this.$gateway.send("wallet", "restore_wallet", this.wallet);
        },
        cancel() {
            this.$router.replace({ path: "/wallet-select" });
        }
    },
    components: {
        OscillateField,
        PasswordStrength
    }
}
</script>

<style>
</style>
