<template>
<q-modal v-model="isVisible" maximized>
    <q-modal-layout>
        <q-toolbar slot="header" color="dark" inverted>
            <q-btn
                flat
                round
                dense
                @click="isVisible = false"
                icon="reply"
                />
            <q-toolbar-title>
                {{ $t("titles.addressDetails") }}
            </q-toolbar-title>
            <q-btn flat @click="isQRCodeVisible = true" :label="$t('buttons.showQRCode')" />
            <q-btn class="q-ml-sm" color="primary" @click="copyAddress()" :label="$t('buttons.copyAddress')" />
        </q-toolbar>

        <div class="layout-padding">

            <template v-if="address != null">

                <AddressHeader :address="address.address"
                               :title="addressHeaderInfo.title"
                               :extra="addressHeaderInfo.extra"
                               :showCopy="false"
                               />


                <template v-if="address.used">
                    <div class="row justify-between" style="max-width: 768px">

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.oscillateBalance") }}</span></div>
                                <div class="value"><span><FormatOscillate :amount="address.balance" /></span></div>
                            </div>
                        </div>

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.oscillateUnlockedBalance") }}</span></div>
                                <div class="value"><span><FormatOscillate :amount="address.unlocked_balance" /></span></div>
                            </div>
                        </div>

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.numberOfUnspentOutputs") }}</span></div>
                                <div class="value"><span>{{ address.num_unspent_outputs }}</span></div>
                            </div>
                        </div>

                    </div>
                </template>
                <template v-else>
                    <div class="row justify-between" style="max-width: 768px">

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.oscillateBalance") }}</span></div>
                                <div class="value"><span>N/A</span></div>
                            </div>
                        </div>

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.oscillateUnlockedBalance") }}</span></div>
                                <div class="value"><span>N/A</span></div>
                            </div>
                        </div>

                        <div class="infoBox">
                            <div class="infoBoxContent">
                                <div class="text"><span>{{ $t("strings.numberOfUnspentOutputs") }}</span></div>
                                <div class="value"><span>N/A</span></div>
                            </div>
                        </div>

                    </div>
                </template>

                <div class="q-mt-sm">

                    <div class="non-selectable">
                        <q-icon name="history" size="24px" />
                        <span class="vertical-middle q-ml-xs">{{ $t("strings.recentIncomingTransactionsToAddress") }}</span>
                    </div>

                    <div style="margin: 0 -16px;">
                        <TxList type="all_in" :limit="5" :to-incoming-address-index="address.address_index" :key="address.address"/>
                    </div>

                </div>

            </template>

        </div>

    </q-modal-layout>

    <template v-if="address != null">
        <q-modal v-model="isQRCodeVisible" minimized :content-css="{padding: '25px'}">

            <div class="text-center q-mb-sm q-pa-md" style="background: white;">
                <qrcode-vue :value="address.address" size="240" ref="qr">
                </qrcode-vue>
                <q-context-menu>
                    <q-list link separator style="min-width: 150px; max-height: 300px;">
                        <q-item v-close-overlay @click.native="copyQR()">
                            <q-item-main :label="$t('menuItems.copyQR')" />
                        </q-item>
                        <q-item v-close-overlay @click.native="saveQR()">
                            <q-item-main :label="$t('menuItems.saveQR')" />
                        </q-item>
                    </q-list>
                </q-context-menu>
            </div>

            <q-btn
                 color="primary"
                 @click="isQRCodeVisible = false"
                 :label="$t('buttons.close')"
             />
        </q-modal>
    </template>

</q-modal>
</template>

<script>
import { mapState } from "vuex"
const { clipboard, nativeImage } = require("electron")
import AddressHeader from "components/address_header"
import FormatOscillate from "components/format_oscillate"
import QrcodeVue from "qrcode.vue";
import TxList from "components/tx_list"
export default {
    name: "AddressDetails",
    computed: mapState({
        addressHeaderInfo (state) {
            if (!this.address) return null

            let title = this.$t('strings.addresses.primaryAddress')
            if (this.address.address_index !== 0) {
                title = this.$t('strings.addresses.subAddress') + ' (' + this.$t('strings.addresses.subAddressIndex', { index: this.address.address_index}) + ')'
            }

            const extra = this.address.used ? this.$t('strings.userUsedAddress') : this.$t('strings.userNotUsedAddress')

            return {
                title,
                extra
            }
        }
    }),
    data () {
        return {
            isVisible: false,
            isQRCodeVisible: false,
            address: null
        }
    },
    methods: {
         copyQR () {
            const data = this.$refs.qr.$el.childNodes[0].toDataURL()
            const img = nativeImage.createFromDataURL(data)
            clipboard.writeImage(img)
             this.$q.notify({
                type: "positive",
                timeout: 1000,
                message: this.$t("notification.positive.qrCopied")
            })
        },
        saveQR() {
            let img = this.$refs.qr.$el.childNodes[0].toDataURL()
            this.$gateway.send("core", "save_png", {img, type: "QR Code"})
        },
        copyAddress() {
            clipboard.writeText(this.address.address)
            this.$q.notify({
                type: "positive",
                timeout: 1000,
                message: this.$t("notification.positive.addressCopied")
            })
        }
    },
    components: {
        AddressHeader,
        TxList,
        FormatOscillate,
        QrcodeVue
    }
}
</script>

<style>
</style>
