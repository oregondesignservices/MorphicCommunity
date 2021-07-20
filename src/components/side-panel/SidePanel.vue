<template>
  <div id="SidePanel">
    <h2 class="accountName">{{ community.name }}</h2>
    <div class="accountInfo">
      <b-link :to="{ name: 'MyCommunity'}" ><b-icon icon="gear-fill" />{{ $t('SidePanel.account-settings_link') }}</b-link>
    </div>


    <!-- member's own bars -->
    <h3 v-t="'SidePanel.own-bars_heading'" />
    <BarsList ref="BarsList"
              :button-attrs="buttonAttrs"
              id="MyMorphicBars"
              :bars="bars"
              :activeBarId="activeBarId"
              :member="currentMember"
              @newbar="newBar(currentMember)"
    />

    <!-- managed members -->
    <template v-if="isManager">
      <h3 v-t="'SidePanel.other-bars_heading'" />
      <MembersList ref="MembersList"
                   id="MembersList"
                   :button-attrs="buttonAttrs"
                   :members="members"
                   :community="community"
                   :activeBarId="activeBarId"
                   :bars="bars"
                   :activeMemberId="activeMemberId"
                   :expandedMembers="expandedMembers"
                   @expandClick="expandClick"
                   @newbar="newBar($event)"
                   @addMember="$event.promise = addMember($event.name)"
      />
    </template>

    <!-- hiding group bars for now -->
    <template v-if="false">
      <!-- group bars -->
      <h3 v-t="'SidePanel.group-bars_heading'" />
      <BarsList ref="BarsList"
                :bars="bars"
                :activeBarId="activeBarId"
                @newBar="newBar()"
      />
    </template>
  </div>
</template>

<style lang="scss">
  @import "~@/styles/_variables.scss";

  #SidePanel {
    font-size: 1rem;
    padding-top: 0.5em;

    h2, h3, h4, h5 {
      font-weight: bold;
      font-size: 1em;
      margin: 1em 0 0.5em 0;
    }

    h2 {
      margin-top: 0;
      font-size: 1.5em;
    }

    h3 {
      margin-top: 3em;
    }

    h4, h5 {
      margin: 4px 2px;
    }

  }

  $indent: 0.3em;

  // non-mobile
  body:not(.isLite) #SidePanel {
    //noinspection CssUnknownTarget
    background: #a5d58a url(/img/bg-green.png) repeat-x bottom;
    min-height: 600px;
    height: 100%;

    .btn-link, a {
      color: black;
    }

    button:not(.btn-link) {
      margin: 1px;
      min-width: 50%;
    }

    a {
      text-decoration: underline;

      &.barLink {
        padding: 0.1em 0.4em;
        font-size: 1em;
        color: black;
        display: block;
      }
    }

    // Indentation
    padding-left: $indent;

    .accountInfo {
      padding-left: $indent * 2;
    }

    .panelBox {
      padding: 0.3em 0 0.3em $indent * 2;

      button {
        margin-left: $indent * 2;
      }
    }

    // Make the list items full width of the panel
    li {
      margin-left: $indent * -3;
      padding-left: $indent * 3;
    }

    .expander .text-link {
      color: $morphic-green-color;
    }

  }

  #SidePanel {

    .memberName {
      display: flex;
      & > :first-child {
        flex-grow: 1;
      }
      .expander {
        margin-right: 0.5em;
      }
    }

    .buttonRow {
      button:first-child {
        margin-right: 1em;
      }

      button {
        font-size: 80%;
      }
    }

    ul {
      margin: 0.5em 0;

      li {
        &.active {
          background-color: $primary-color;

          .barLink {
            color: white !important;
          }
        }

        // Custom bullet to reduce the space after the bullet.
        a::before {
          content: "\2022";
          color: black;
          text-decoration: none !important;
          display: inline-block;
          font-size: 1em;
          transform: scale(1.3);
          margin-right: $indent * 1.5;
        }
      }
    }

    $panelBox-color: #85C399;
    .panelBox {
      overflow: hidden;
      .expandable {
        color: black;
      }
      .memberName {
        margin: 0;
        padding: 0.2em 0;
      }

      font-size: 0.95em;
      border-radius: 8px;
      margin-top: 0.5em;
      margin-right: $indent;
      background-color: $panelBox-color;

      &:last-of-type {
        margin-bottom: 0.5em;
        background-color: red !important;
      }

      &.active {
        background-color: lighten($panelBox-color, 7%);
      }

      & > .expandable {
        font-weight: bold;
      }
    }
  }

  // Mobile overrides
  body.isLite #SidePanel {
    font-size: 1.2rem;
    padding: 1em;
    //noinspection CssUnknownTarget
    background: unset;
    .accountName {
      font-size: 1.1em;
      font-weight: normal;
    }

    h2, h3, h4, h5 {
      font-weight: bold;
      font-size: 1.1em;
    }
    h3 {
      margin-top: 1em;
    }

    li > a {
      display: block;
    }

    .expander {
      right: 2em;
    }
    .expander-background {
      color: $link-color-darker;
    }

    .panelBox {
      padding: 0.5em;
      margin-right: 0;;
      &:nth-child(even) {
        background-color: #e0f1d7;
      }
      &:nth-child(odd) {
        background-color: #bfdfd0;
      }
    }

    button:not(.btn-link) {
      padding-left: 1.5em;
      padding-right: 1.5em;
    }

  }

  #MembersList > div:last-child {
    margin-top: 0.5em;
  }
</style>

<script>

import MembersList from "@/components/side-panel/MembersList";
import BarsList from "@/components/side-panel/BarsList";
import * as Bar from "@/utils/bar";
import * as communityService from "@/services/communityService";
import { membersMixin } from "@/mixins/members.js";

export default {
    name: "SidePanel",
    components: {
        MembersList,
        BarsList
    },
    mixins: [membersMixin],
    props: {
        /** @type {Community} */
        community: Object,
        /**
         * All bars in the community
         * @type {Array<BarDetails>}
         */
        bars: Array,
        /** @type {Array<CommunityMember>} */
        members: Array,
        activeBarId: String,
        activeMemberId: String
    },
    mounted() {
        this.$el.querySelectorAll("button.collapseAll").forEach(b => {
            b.disabled = true;
        });
    },
    data() {
        return {
            /** @type {GUID} The member that was just added */
            newestMemberId: null
        };
    },
    computed: {
        /**
         * Get the bars owned by the current user.
         * @return {Array<BarDetails>} The bars owned by the member.
         */
        memberBars() {
            return Bar.getMemberBars(this.bars, this.currentMember);
        },
        hasOwnBar() {
            return this.memberBars.length > 0;
        },
        /**
         * @return {CommunityMember} The current member
         */
        currentMember() {
            return this.members.find(m => m.isCurrent) || {};
        },
        isManager() {
            return this.currentMember.role === "manager";
        },
        expandedMembers() {
            return [this.newestMemberId];
        },
        buttonAttrs() {
            return this.isLite
                ? {
                    variant: "secondary",
                    size: "md"
                }
                : {
                    variant: "primary",
                    size: "sm"
                };
        }
    },
    methods: {
        reloadAll() {
            this.$emit("reload");
        },
        /**
         * Toggles a collapsable element (or all collapsable child elements).
         * @param {Element|String} el The element, or the expand-group to toggle all.
         * @param {Boolean?} toggle true/false to expand/collapse, omit to toggle.
         */
        expandClick(el, toggle) {
            const all = typeof(el) === "string";
            const expandGroup = all ? el : el.getAttribute("expand-group");

            const e = all
                ? this.$el.querySelectorAll(`.expandable[expand-group=${expandGroup}]`)
                : [el];

            e.forEach(e => e.classList.toggle("expanded", toggle));

            // Disable the expand/collapse all buttons, if required
            if (expandGroup) {
                const expandAllButton = this.$el.querySelector(`button[expand-group=${expandGroup}].expandAll`);
                const collapseAllButton = this.$el.querySelector(`button[expand-group=${expandGroup}].collapseAll`);

                if (expandAllButton || collapseAllButton) {
                    const someExpanded = this.$el.querySelectorAll(`.expandable.expanded[expand-group=${expandGroup}]`).length;
                    const someCollapsed = this.$el.querySelectorAll(`.expandable:not(.expanded)[expand-group=${expandGroup}]`).length;

                    expandAllButton.disabled = !someCollapsed;
                    collapseAllButton.disabled = !someExpanded;
                }
            }
        },

        /**
         * Called when a component fires the newbar event.
         * @param {CommunityMember} [member] The member this bar belongs to (omit for a shared bar).
         * @param {String} [name] The name of the bar.
         * @return {Promise<BarDetails>} Resolves when the new bar has been created.
         */
        newBar(member, name) {
            return this.createBar(member, name).then(bar => {
                this.showMessage("New bar created");

                const route = Bar.getUserBarEditRoute(bar.id, member);
                this.$router.push(route);

                return bar;
            });
        },

        /**
         * Creates a new bar
         * @param {CommunityMember} [member] The member this bar belongs to (omit for a shared bar).
         * @param {String} [name] The name of the bar.
         * @return {Promise<BarDetails>} Resolves when the new bar has been created.
         */
        async createBar(member, name) {
            let barName = name || (member ? `Bar for ${member.fullName}` : undefined);
            if (barName === "(no name)") {
                barName = "My MorphicBar";
            }

            const existingNames = this.bars.filter(b => member.bar_ids.includes(b.id)).map(b => b.name);
            let suffixNumber = 1;
            let newBarName = barName;
            while (existingNames.includes(newBarName)) {
                newBarName = `${barName} (${suffixNumber})`;
                suffixNumber++;
            }

            const bar = Bar.newBar(!member, newBarName);

            const createResponse = await communityService.createCommunityBar(this.communityId, bar);
            bar.id = createResponse.data.bar.id;

            if (member) {
                await this.memberAddBar(bar, member);
            }

            return bar;
        },

        /**
         * Adds a new member, and creates a new bar for them.
         * @param {String} name The new member's name.
         * @return {Promise<CommunityMember>} Resolves with the community member.
         */
        async addMember(name) {
            // create the new member
            const addResult = await communityService.addCommunityMember(this.communityId, {
                first_name: name,
                last_name: ""
            });

            /** @type {CommunityMember} */
            const member = addResult.data.member;

            // Create a new bar for the member
            const bar = await this.createBar(member);

            this.newestMemberId = member.id;
            this.reloadAll();
            this.showMessage("New member added");

            const route = Bar.getUserBarEditRoute(bar.id, member);
            this.$router.push(route);

            return member;
        }
    }
};
</script>
