<template>
<q-page class="address-book">

    <div class="header row q-pt-md q-pb-xs q-mx-md q-mb-none items-center non-selectable">
            {{ $t("titles.addressBook") }}
    </div>

    <template v-if="address_book_combined.length">
        <q-list link no-border :dark="theme=='dark'" class="oscillate-list">
            <q-item class="oscillate-list-item" v-for="(entry, index) in address_book_combined" @click.native="details(entry)" :key="`${entry.address}-${entry.name}`">
                <q-item-main>
                    <q-item-tile class="ellipsis" label>{{ entry.address }}</q-item-tile>
                    <q-item-tile sublabel class="non-selectable">{{ entry.name }}</q-item-tile>
                </q-item-main>
                <q-item-side>
                    <q-icon size="24px" :name="entry.starred ? 'star' : 'star_border'" />
                    <q-btn
                        color="secondary"
                        style="margin-left: 10px;"
                        :label="$t('buttons.send')"
                        :disabled="view_only"
                        @click="sendToAddress(entry, $event)"
                    />
                </q-item-side>

                <q-context-menu>
                    <q-list link separator style="min-width: 150px; max-height: 300px;">
                        <q-item v-close-overlay
                                @click.native="details(entry)">
                            <q-item-main :label="$t('menuItems.showDetails')" />
                        </q-item>

                        <q-item v-close-overlay
                                @click.native="sendToAddress(entry, $event)">
                            <q-item-main :label="$t('menuItems.sendToThisAddress')" />
                        </q-item>

                        <q-item v-close-overlay
                                @click.native="copyAddress(entry, $event)">
                            <q-item-main :label="$t('menuItems.copyAddress')" />
                        </q-item>
                    </q-list>
                </q-context-menu>

            </q-item>
        </q-list>
    </template>
    <template v-else>
        <p class="q-ma-md">{{ $t("strings.addressBookIsEmpty") }}</p>
    </template>

    <q-page-sticky position="bottom-right" :offset="[18, 18]">
        <q-btn
            :disable="!is_ready"
            round
            color="primary"
            @click="addEntry"
            icon="add"
            />
    </q-page-sticky>
    <AddressBookDetails ref="addressBookDetails" />

</q-page>
</template>

<script>
const { clipboard } = require("electron")
import { mapState } from "vuex"
import Identicon from "components/identicon"
import AddressBookDetails from "components/address_book_details"
export default {
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        view_only: state => state.gateway.wallet.info.view_only,
        address_book: state => state.gateway.wallet.address_list.address_book,
        address_book_starred: state => state.gateway.wallet.address_list.address_book_starred,
        is_ready (state) {
            return this.$store.getters["gateway/isReady"]
        },
        address_book_combined (state) {
            const starred = this.address_book_starred.map(a => ({ ...a, starred: true }));
            return [
                ...starred,
                ...this.address_book
            ]
        }
    }),
    methods: {
        details: function (entry) {
            this.$refs.addressBookDetails.entry = entry
            this.$refs.addressBookDetails.mode = "view"
            this.$refs.addressBookDetails.isVisible = true
        },
        addEntry: function () {
            this.$refs.addressBookDetails.entry = null
            this.$refs.addressBookDetails.mode = "new"
            this.$refs.addressBookDetails.isVisible = true
        },
        sendToAddress (address, event) {
            event.stopPropagation()
            for(let i = 0; i < event.path.length; i++) {
                if(event.path[i].tagName == "BUTTON") {
                    event.path[i].blur()
                    break
                }
            }
            this.$router.replace({ path: "send", query: {address: address.address, payment_id: address.payment_id} });
        },
        copyAddress (entry, event) {
            event.stopPropagation()
            for(let i = 0; i < event.path.length; i++) {
                if(event.path[i].tagName == "BUTTON") {
                    event.path[i].blur()
                    break
                }
            }
            clipboard.writeText(entry.address)
            if(entry.payment_id) {
                this.$q.dialog({
                    title: this.$t("dialog.copyAddress.title"),
                    message: this.$t("dialog.copyAddress.message"),
                    ok: {
                        label: this.$t("dialog.buttons.ok")
                    },
                }).catch(() => null).then(password => {
                    this.$q.notify({
                        type: "positive",
                        timeout: 1000,
                        message: this.$t("notification.positive.addressCopied")
                    })
                })
            } else {
                this.$q.notify({
                    type: "positive",
                    timeout: 1000,
                    message: this.$t("notification.positive.addressCopied")
                })
            }
        }

    },
    components: {
        Identicon,
        AddressBookDetails
    }
}
</script>

<style lang="scss">
.address-book {
    .header {
        font-size: 14px;
        font-weight: 500
    }

    .oscillate-list-item {
        cursor: pointer;
        padding-top: 12px;
        padding-bottom: 12px;

        .q-item-sublabel  {
            font-size: 14px;
        }

        .q-item-label {
            font-weight: 400;
        }

        .q-item-side {
            display: flex;
            justify-content: center;
            align-items: center;
            margin-left: 12px;
        }
    }

}
</style>
