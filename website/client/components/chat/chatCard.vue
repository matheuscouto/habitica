<template lang="pug">
div
  .mentioned-icon(v-if='isUserMentioned')
  .message-hidden(v-if='msg.flagCount === 1 && user.contributor.admin') Message flagged once, not hidden
  .message-hidden(v-if='msg.flagCount > 1 && user.contributor.admin') Message hidden
  .card-body
    h3.leader(
      :class='userLevelStyle(msg)',
      @click="showMemberModal(msg.uuid)",
      v-b-tooltip.hover.top="tierTitle",
      v-if="msg.user"
    )
      | {{msg.user}}
      .svg-icon(v-html="tierIcon")
    p.time
      span.mr-1(v-if="msg.username") @{{ msg.username }}
      span.mr-1(v-if="msg.username") •
      span(v-b-tooltip="", :title="msg.timestamp | date") {{ msg.timestamp | timeAgo }}
    .text(v-html='atHighlight(parseMarkdown(msg.text))')
    hr
    .d-flex(v-if='msg.id')
      .action.d-flex.align-items-center(v-if='!inbox', @click='copyAsTodo(msg)')
        .svg-icon(v-html="icons.copy")
        div {{$t('copyAsTodo')}}
      .action.d-flex.align-items-center(v-if='!inbox && user.flags.communityGuidelinesAccepted && msg.uuid !== "system"', @click='report(msg)')
        .svg-icon(v-html="icons.report")
        div {{$t('report')}}
        // @TODO make flagging/reporting work in the inbox. NOTE: it must work even if the communityGuidelines are not accepted and it MUST work for messages that you have SENT as well as received. -- Alys
      .action.d-flex.align-items-center(v-if='msg.uuid === user._id || inbox || user.contributor.admin', @click='remove()')
        .svg-icon(v-html="icons.delete")
        | {{$t('delete')}}
      .ml-auto.d-flex(v-b-tooltip="{title: likeTooltip(msg.likes[user._id])}", v-if='!inbox')
        .action.d-flex.align-items-center.mr-0(@click='like()', v-if='likeCount > 0', :class='{active: msg.likes[user._id]}')
          .svg-icon(v-html="icons.liked", :title='$t("liked")')
          | +{{ likeCount }}
        .action.d-flex.align-items-center.mr-0(@click='like()', v-if='likeCount === 0', :class='{active: msg.likes[user._id]}')
          .svg-icon(v-html="icons.like", :title='$t("like")')
          span(v-if='!msg.likes[user._id]') {{ $t('like') }}
</template>

<style lang="scss">
  .at-highlight {
    background-color: rgba(213, 200, 255, 0.32);
    padding: 0.1rem;
  }

  .at-text {
    color: #6133b4;
  }
</style>

<style lang="scss" scoped>
  @import '~client/assets/scss/colors.scss';
  @import '~client/assets/scss/tiers.scss';

  .mentioned-icon {
    width: 16px;
    height: 16px;
    border-radius: 50%;
    background-color: #bda8ff;
    box-shadow: 0 1px 1px 0 rgba(26, 24, 29, 0.12);
    position: absolute;
    right: -.5em;
    top: -.5em;
  }

  .message-hidden {
    margin-left: 1.5em;
    margin-top: 1em;
    color: red;
  }

  hr {
    margin-bottom: 0.5rem;
    margin-top: 0.5rem;
  }

  .card-body {
    padding: 0.75rem 1.25rem 0.75rem 1.25rem;

    .leader {
      margin-bottom: 0;
    }

    h3 { // this is the user name
      cursor: pointer;
      display: inline-block;
      font-size: 16px;

      .svg-icon {
        width: 10px;
        display: inline-block;
        margin-left: .5em;
      }
    }

    .time {
      font-size: 12px;
      color: #878190;
      margin-bottom: 0.5rem;
    }

    .text {
      font-size: 14px;
      color: #4e4a57;
      text-align: left !important;
      min-height: 0rem;
      margin-bottom: -0.5rem;
    }
  }

  .action {
    display: inline-block;
    color: #878190;
    margin-right: 1em;
    font-size: 12px;

    :hover {
      cursor: pointer;
    }

    .svg-icon {
      color: #A5A1AC;
      margin-right: .2em;
      width: 16px;
    }
  }

  .active {
    color: $purple-300;

    .svg-icon {
      color: $purple-400;
    }
  }
</style>

<script>
import axios from 'axios';
import moment from 'moment';
import cloneDeep from 'lodash/cloneDeep';
import escapeRegExp from 'lodash/escapeRegExp';
import max from 'lodash/max';

import habiticaMarkdown from 'habitica-markdown';
import { mapState } from 'client/libs/store';
import styleHelper from 'client/mixins/styleHelper';

import achievementsLib from '../../../common/script/libs/achievements';

import deleteIcon from 'assets/svg/delete.svg';
import copyIcon from 'assets/svg/copy.svg';
import likeIcon from 'assets/svg/like.svg';
import likedIcon from 'assets/svg/liked.svg';
import reportIcon from 'assets/svg/report.svg';
import tier1 from 'assets/svg/tier-1.svg';
import tier2 from 'assets/svg/tier-2.svg';
import tier3 from 'assets/svg/tier-3.svg';
import tier4 from 'assets/svg/tier-4.svg';
import tier5 from 'assets/svg/tier-5.svg';
import tier6 from 'assets/svg/tier-6.svg';
import tier7 from 'assets/svg/tier-7.svg';
import tier8 from 'assets/svg/tier-mod.svg';
import tier9 from 'assets/svg/tier-staff.svg';
import tierNPC from 'assets/svg/tier-npc.svg';

export default {
  props: ['msg', 'inbox', 'groupId'],
  mixins: [styleHelper],
  data () {
    return {
      icons: Object.freeze({
        like: likeIcon,
        copy: copyIcon,
        report: reportIcon,
        delete: deleteIcon,
        liked: likedIcon,
        tier1,
        tier2,
        tier3,
        tier4,
        tier5,
        tier6,
        tier7,
        tier8,
        tier9,
        tierNPC,
      }),
    };
  },
  filters: {
    timeAgo (value) {
      return moment(value).fromNow();
    },
    date (value) {
      // @TODO: Vue doesn't support this so we cant user preference
      return moment(value).toDate();
    },
  },
  computed: {
    ...mapState({user: 'user.data'}),
    isUserMentioned () {
      const message = this.msg;
      const user = this.user;

      if (message.hasOwnProperty('highlight')) return message.highlight;

      message.highlight = false;
      const messageText = message.text.toLowerCase();
      const displayName = user.profile.name;
      const username = user.auth.local && user.auth.local.username;
      const mentioned = max([messageText.indexOf(username.toLowerCase()), messageText.indexOf(displayName.toLowerCase())]);
      if (mentioned === -1) return message.highlight;

      const escapedDisplayName = escapeRegExp(displayName);
      const escapedUsername = escapeRegExp(username);
      const pattern = `@(${escapedUsername}|${escapedDisplayName})(\\b)`;
      const precedingChar = messageText.substring(mentioned - 1, mentioned);
      if (mentioned === 0 || precedingChar.trim() === '' || precedingChar === '@') {
        let regex = new RegExp(pattern, 'i');
        message.highlight = regex.test(messageText);
      }

      return message.highlight;
    },
    likeCount () {
      const message = this.msg;
      if (!message.likes) return 0;

      let likeCount = 0;
      for (let key in message.likes) {
        let like = message.likes[key];
        if (like) likeCount += 1;
      }
      return likeCount;
    },
    tierIcon () {
      const message = this.msg;
      const isNPC = Boolean(message.backer && message.backer.npc);
      if (isNPC) {
        return this.icons.tierNPC;
      }
      return this.icons[`tier${message.contributor.level}`];
    },
    tierTitle () {
      const message = this.msg;
      return achievementsLib.getContribText(message.contributor, message.backer) || '';
    },
  },
  methods: {
    async like () {
      let message = cloneDeep(this.msg);

      await this.$store.dispatch('chat:like', {
        groupId: this.groupId,
        chatId: message.id,
      });

      if (!message.likes[this.user._id]) {
        message.likes[this.user._id] = true;
      } else {
        message.likes[this.user._id] = !message.likes[this.user._id];
      }

      this.$emit('message-liked', message);
      this.$root.$emit('bv::hide::tooltip');
    },
    likeTooltip (likedStatus) {
      if (!likedStatus) return this.$t('like');
    },
    copyAsTodo (message) {
      this.$root.$emit('habitica::copy-as-todo', message);
    },
    async report () {
      this.$root.$emit('habitica::report-chat', {
        message: this.msg,
        groupId: this.groupId,
      });
    },
    async remove () {
      if (!confirm(this.$t('areYouSureDeleteMessage'))) return;

      const message = this.msg;
      this.$emit('message-removed', message);

      if (this.inbox) {
        await axios.delete(`/api/v4/inbox/messages/${message.id}`);
        return;
      }

      await this.$store.dispatch('chat:deleteChat', {
        groupId: this.groupId,
        chatId: message.id,
      });
    },
    showMemberModal (memberId) {
      this.$emit('show-member-modal', memberId);
    },
    atHighlight (text) {
      const userRegex = new RegExp(`@(${this.user.auth.local.username}|${this.user.profile.name})(?:\\b)`, 'gi');
      const atRegex = new RegExp(/(?!\b)@[\w-]+/g);

      if (userRegex.test(text)) {
        text = text.replace(userRegex, match => {
          return `<span class="at-highlight at-text">${match}</span>`;
        });
      }

      if (atRegex.test(text)) {
        text = text.replace(atRegex, match => {
          return `<span class="at-text">${match}</span>`;
        });
      }

      return text;
    },
    parseMarkdown (text) {
      return habiticaMarkdown.render(text);
    },
  },
};
</script>
