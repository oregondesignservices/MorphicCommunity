<template>
  <div>
    <!-- MODALs: BEGIN -->
    <EditButtonDialog ref="editDialog" :bar="barDetails" />
    <CopyBarDialog id="copyBarDialog" :bars="barsList" :members="membersList" :current-bar="barDetails" @change="onBarChanged" />

    <b-modal id="unsavedChanges" title="Unsaved Changes">
      <p>You have made some changes to this bar, but they are not saved yet.</p>
      <p>If you leave now, the recent modifications will be lost.</p>

      <template #modal-footer="{ ok, cancel, hide }">
        <b-button @click="cancel()" variant="secondary">Cancel</b-button>
        <b-button @click="hide('no-save')" variant="secondary">Continue, without saving</b-button>
        <b-button @click="hide('save')" variant="primary">Save</b-button>
      </template>
    </b-modal>
    <!-- MODALs: END -->

    <!-- EDITOR v2 -->
    <div id="EditorContainer">
      <div v-if="!isLite" class="editorSide left">
        <SidePanel :community="community" :bars="barsList" :members="membersList" :activeMemberId="activeMemberId" :activeBarId="barDetails.id" @reload="loadAllData()" />
      </div>
      <div v-show="editorVisible" class="editor">
        <div id="barEditor" class="pt-2">

          <b-link v-if="isLite" to="/">
            <b-img src="/img/back-arrow.svg" width="20" height="20" class="switchBarIcon" />Switch Bar or Person
          </b-link>

          <!-- Bar info, actions, and editor tabs -->
          <EditorDetails ref="EditorDetails"
                         :bar-details="barDetails"
                         :bars-list="barsList"
                         :bar-members="barMembers"
                         :is-changed="isChanged"
                         :member-details="memberDetails"
                         :new-bar="newBar"
                         @save-bar="saveBar()" @revert-bar="revertBar()"
          />

          <!-- the focus/mobile editor -->
          <LiteBarEditor v-if="isLite"
                         ref="LiteBarEditor"
                         :bar-details="barDetails"
                         @edit-item="showEditDialog($event)"
                         @add-item="showCatalog(true)"
          />


          <!-- the desktop editor -->
          <DesktopBarEditor v-else
                            ref="DesktopBarEditor"
                            :bar-details="barDetails"
                            :is-changed="isChanged"
                            @edit-item="showEditDialog($event)"
                            @bar-changed="onBarChanged"
                            @click="$refs.EditorDetails.closeTab()"
                            @item-dropped="addBarItem($event.item, $event.noImage, $event.index)"
          />
        </div>
      </div>

      <!-- Button Catalogue -->
      <div v-show="catalogVisible" :class="isLite && 'p-3'" class="editorSide right">

        <template v-if="isLite">
          <h1>Button Catalog: Buttons you can add</h1>
          <div class="mb-3">
            <b-button @click="showCatalog(false)">Cancel</b-button>
          </div>
        </template>


        <ButtonCatalog ref="ButtonCatalog"
                       :button-catalog="buttonCatalog"
                       @item-selected="addBarItem($event.item, $event.noImage)"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss">
  #barEditor {
    padding-left: 15px;
    padding-right: 15px;
  }

.switchBarIcon {
  vertical-align: top;
}

body:not(.isLite) {
  #EditorContainer {
    display: flex;

    .editor {
      flex-grow: 1;
      flex-shrink: 1;
    }

    .editorSide {
      flex-shrink: 0;
      width: 15em;
      min-width: 16.66%;
    }
  }
}
</style>

<script>

import SidePanel from "@/components/side-panel/SidePanel";
import {
    createCommunityBar,
    getCommunity,
    getCommunityBar,
    getCommunityBars,
    getCommunityMembers,
    updateCommunityBar
} from "@/services/communityService";
import { buttonCatalog, MESSAGES } from "@/utils/constants";
import { predefinedBars } from "@/utils/predefined";
import * as Bar from "@/utils/bar";
import EditButtonDialog from "@/components/editor/EditButtonDialog";
import ButtonCatalog from "@/components/editor/ButtonCatalog";
import CopyBarDialog from "@/components/dialogs/CopyBarDialog";
import EditorDetails from "@/components/editor/EditorDetails";
import DesktopBarEditor from "@/components/editor/DesktopBarEditor";
import LiteBarEditor from "@/components/editor/LiteBarEditor";

export default {
    name: "MorphicBarEditor",
    components: {
        LiteBarEditor,
        DesktopBarEditor,
        EditorDetails,
        ButtonCatalog,
        CopyBarDialog,
        EditButtonDialog,
        SidePanel
    },
    props: {
        catalogView: Boolean
    },
    methods: {

        /**
         * Add an item to the bar.
         * @param {BarItem} catalogButton The new button, from the catalog.
         * @param {Boolean} [noImage] True if the button shall have no image.
         * @param {Number} [insertAt] The index of the new button.
         */
        addBarItem: function (catalogButton, noImage, insertAt) {
            /** @type {BarItem} */
            const barItem = Bar.addItem(this.barDetails, catalogButton, insertAt);
            if (noImage) {
                barItem.configuration.image_url = "";
            }

            // close the catalog
            this.showCatalog(false);
            this.onBarChanged();

            // Show the edit dialog if it has parameters, a placeholder, or has errors.
            let showEdit;
            if (barItem.data.showEdit === undefined) {
                showEdit = barItem.data.paramFields.length > 1 || barItem.data.isPlaceholder || barItem.data.hasError;
            } else {
                showEdit = barItem.data.showEdit;
            }

            if (showEdit) {
                this.showEditDialog(barItem);
            }
        },
        /**
         * Called when the bar changes, after it is loaded.
         */
        onBarChanged: function () {
            this.isChanged = true;
            this.storeUnsavedBar();
        },

        storeUnsavedBar: function () {
            Bar.checkBar(this.barDetails);
            this.$store.dispatch("unsavedChanges", this.isChanged);
            this.$store.dispatch("unsavedBar", this.isChanged && this.barDetails);
        },

        revertBar: function () {
            if (window.confirm("Are you sure you want reload last saved version of the bar? This means you will lose all unsaved changes!")) {
                this.isChanged = false;
                this.barDetails = JSON.parse(JSON.stringify(this.originalBarDetails));
            }
        },

        loadAllData: function () {
            this.loadBarData();
            this.loadBarMembers();
            this.getCommunityData();
        },

        /** Loads the initial bar data */
        loadBarData: async function () {
            var barId = this.$route.query.barId;

            // If there is a bar unsaved, redirect to that one instead.
            /** @type {BarDetails} */
            const unsavedBar = this.$store.getters.unsavedBar;
            if (unsavedBar && unsavedBar.id !== barId) {
                const barName = Bar.getBarName(unsavedBar);
                const message = `There is already a bar (${barName}) that has unsaved changes. The recent changes will be lost if you continue.`;
                const ok = await this.showConfirm(message,
                    ["Continue", `Go to ${barName}`],
                    "Unsaved Changes",
                    {
                        cancelVariant: "primary"
                    }
                );

                if (!ok) {
                    this.$router.push(this.getBarEditRoute(unsavedBar));
                    return;
                }
            }

            if (unsavedBar && unsavedBar.id === barId) {
                this.barDetails = unsavedBar;
                this.onBarChanged();
            } else {
                this.isChanged = false;
                this.storeUnsavedBar();
                // Load a saved bar.
                getCommunityBar(this.communityId, barId)
                    .then(resp => {
                        this.barDetails = resp.data;
                        this.updateOriginalBarDetails();
                    })
                    .catch(err => {
                        console.error(err);
                    });
            }
        },
        // hack to refresh css rendering due to bars being fucked up in their CSS
        refreshBar() {
            const myref = this.$refs.myref;
            if (myref) {
                myref.classList.toggle("minWidth1px");
            }
        },
        updateOriginalBarDetails: function () {
            this.originalBarDetails = JSON.parse(JSON.stringify(this.barDetails));
        },
        addBar: function () {
            this.onSave = true;
            const data = this.barDetails;
            data.is_shared = true;
            // const drawerItems = this.drawerItems.concat(this.drawerItemsSecond)
            // data.items = this.primaryItems.concat(drawerItems)

            createCommunityBar(this.communityId, data)
                .then((resp) => {
                    if (resp.status === 200) {
                        this.showMessage(MESSAGES.barAdded);
                        this.isChanged = false;
                        this.updateOriginalBarDetails();

                        this.$router.push("/dashboard");
                    }
                })
                .catch(err => {
                    console.error(err);
                });
        },

        /**
         * Saves the bar.
         * @return {Promise} Resolves when complete.
         */
        saveBar: function () {
            this.onSave = true;
            const data = this.barDetails;
            // const drawerItems = this.drawerItems.concat(this.drawerItemsSecond)
            // data.items = this.primaryItems.concat(drawerItems)

            // Set the image_path for the items, for images which the client does not have locally.
            data.items.forEach(item => {
                item.configuration.image_path = this.getIconUrl(item.configuration.image_url);
            });

            return updateCommunityBar(this.communityId, this.$route.query.barId, data)
                .then((resp) => {
                    if (resp.status === 200) {
                        this.showMessage(MESSAGES.barUpdated);
                        this.isChanged = false;
                        this.updateOriginalBarDetails();
                    }
                });
        },

        /**
         * Shows the edit button dialog.
         * @param {BarItem} [item] The item to edit.
         */
        showEditDialog: function (item) {
            this.editDialog.showDialog(item).then(changed => {
                Bar.checkBar(this.barDetails);
                if (changed) {
                    this.onBarChanged();
                }
                this.$forceUpdate();
            });
        },

        /**
         * Lite mode: Sets the visibility of the button catalog.
         * @param {Boolean} show true to show, false to hide.
         */
        showCatalog: function (show) {
            const catalogRoute = {...this.$route };
            catalogRoute.query = {...catalogRoute.query };
            const lastValue = catalogRoute.query.catalogView;

            if (show) {
                catalogRoute.query.catalogView = true;
            } else {
                delete catalogRoute.query.catalogView;
            }

            if (lastValue !== catalogRoute.query.catalogView) {
                this.$router.push(catalogRoute);
            }

            this.$refs.ButtonCatalog.$el.focus();

        },

        loadBarMembers: function () {
            getCommunityBars(this.communityId)
                .then(resp => {
                    const barsData = resp.data.bars;
                    getCommunityMembers(this.communityId)
                        .then((resp) => {
                            this.barsList = barsData;
                            this.membersList = resp.data.members;
                        })
                        .catch(err => {
                            console.error(err);
                        });
                })
                .catch(err => {
                    console.error(err);
                });
        },
        getCommunityData: function () {
            getCommunity(this.communityId)
                .then((community) => {
                    this.community = community.data;
                })
                .catch(err => {
                    console.error(err);
                });
        },
        /**
         * Update the filtered arrays of bars.
         */
        updateBarLists() {
            this.sharedBars = [];
            this.userBars = [];
            this.barsList.forEach(bar => {
                const list = bar.is_shared
                    ? this.sharedBars
                    : this.userBars;
                list.push(bar);
            });
        },

        /**
         * Gets the edit route for a bar.
         * @param {BarDetails} bar The bar.
         * @return {Object} The route.
         */
        getBarEditRoute: function (bar) {
            let route;
            if (!bar.is_shared) {
                var member = this.membersList.find(m => m.bar_ids.includes(bar.id));
                route = member && Bar.getUserBarEditRoute(bar.id, member);
            }

            return route || Bar.getBarEditRoute(bar);
        },
        /**
         * Confirms if the user wants to leave the page, if there have been unsaved changes
         * @param {Route} to The target route.
         * @param {Route} from The current route.
         * @return {Promise<Boolean>} Resolves with true to move to the next page.
         */
        leavePage: async function (to, from) {
            var nextPage = true;
            const sameBar = (to.query.barId === from.query.barId && to.query.memberId === from.query.memberId);

            if (!sameBar && this.isChanged) {
                const dialogResult = await this.showModalDialog("unsavedChanges");

                switch (dialogResult) {
                case "save":
                    await this.saveBar();
                    break;
                case "no-save":
                    break;
                default:
                    nextPage = false;
                    break;
                }
            }

            if (nextPage) {
                // Discard the current bar, if moving to another page.
                this.isChanged = false;
                this.storeUnsavedBar();
            }

            return nextPage;
        }
    },
    computed: {
        /**
         * Gets the members which this bar belongs to.
         * @return {Array<CommunityMember>} The members this bar belongs to, empty array for shared bars.
         */
        barMembers: function () {
            return this.barDetails.is_shared
                ? []
                : this.membersList.filter(m => m.bar_ids.includes(this.barDetails.id)).sort((a, b) => b.isCurrent);
        },
        activeMemberId: function () {
            var memberId;
            if (this.barMembers.length === 1) {
                memberId = this.barMembers[0].id;
            } else if (this.barMembers.some(m => m.id === this.$route.query.memberId)) {
                memberId = this.$route.query.memberId;
            }

            return memberId;
        },

        /**
         * The details of the selected member, as identified by activeMemberId
         * @return {CommunityMember|null} The member.
         */
        memberDetails: function () {
            return this.activeMemberId && this.membersList.find(m => m.id === this.activeMemberId);
        },

        editDialog: function () { return this.$refs.editDialog; },

        /**
         * @return {DesktopBarEditor|LiteBarEditor} The bar editor.
         */
        barEditor: function () {
            return this.$refs.DesktopBarEditor || this.$refs.LiteBarEditor;
        },

        editorVisible: function () {
            return !(this.isLite && this.catalogVisible);
        },
        catalogVisible: function () {
            return !this.isLite || this.catalogView;
        }

    },
    mounted() {
        this.loadAllData();
    },
    watch: {
        "memberDetails.id": function (newValue, oldValue) {
            this.updateBarLists();
        },
        barsList: function (newValue) {
            this.updateBarLists();
        },
        makeAButtons: function (newValue, oldValue) {
            if (!this.dragMakeAButton) {
                this.makeAButtons = oldValue;
                this.dragMakeAButton = true;
            }
        },
        predefinedButtons: function (newValue, oldValue) {
            if (!this.dragPredefinedButton) {
                this.predefinedButtons = oldValue;
                this.dragPredefinedButton = true;
            }
        },
        drawerItemsSecond: function (newValue, oldValue) {
            if (oldValue.length !== newValue.length) {
                this.isChanged = true;
            }
            let item = {};
            if (newValue && newValue.length > 0) {
                for (let i = 0; i < newValue.length; i++) {
                    if (newValue[i].is_primary === true) {
                        item = newValue[i];
                        item.is_primary = false;
                    }
                }
            }
        },
        isChanged: function () {
            this.storeUnsavedBar();
        },
        "$route.query": function () {
            // If the bar has changed, reload everything.
            if (this.$route.query.barId !== this.barDetails.id) {
                this.loadAllData();
            }
        }
    },
    async beforeRouteUpdate(to, from, next) {
        const proceed = this.$store.getters.isLoggedIn ? await this.leavePage(to, from) : true;
        next(proceed);
    },
    async beforeRouteLeave(to, from, next) {
        const proceed = this.$store.getters.isLoggedIn ? await this.leavePage(to, from) : true;
        next(proceed);
    },
    beforeUpdate() {
        this.refreshBar();
    },
    data() {
        return {
            // messages
            leavePageMessage: MESSAGES.leavePageAlert,
            // flags
            newBar: false,
            isChanged: false,
            editBarName: false,
            onSave: false,

            // data for the community manager
            community: {},
            /**
             * All bars
             * @type {Array<BarDetails>} */
            barsList: [],
            /**
             * Group Bars
             * @type {Array<BarDetails>}
             */
            sharedBars: [],
            /**
             * User specific bars.
             * @type {Array<BarDetails>}
             */
            userBars: [],
            /** All community members
             * @type {Array<CommunityMember>} */
            membersList: [],

            /** @type {ButtonCatalog} Button catalog. */
            buttonCatalog: buttonCatalog,

            /** @type {BarDetails} */
            barDetails: {},
            originalBarDetails: {},

            newBarDetails: {
                name: "New Bar",
                is_shared: false,
                items: []
            },
            predefinedBars: predefinedBars
        };
    }
};
</script>
