<template>
    <div
        :key="'messagelist-' + buffer.name"
        v-resizeobserver="onListResize"
        class="kiwi-messagelist"
        :class="{'kiwi-messagelist--smoothscroll': smooth_scroll}"
        @click.self="onListClick"
    >
        <div v-resizeobserver="onListResize">
            <div
                v-if="shouldShowChathistoryTools"
                class="kiwi-messagelist-scrollback"
            >
                <a
                    v-if="!buffer.flag('is_requesting_chathistory')"
                    class="u-link"
                    @click="buffer.requestScrollback()"
                >
                    {{ $t('messages_load') }}
                </a>
                <a v-else class="u-link">...</a>
            </div>

            <div v-for="day in filteredMessagesGroupedDay" :key="day.dayNum">
                <div
                    v-if="filteredMessagesGroupedDay.length > 1 && day.messages.length > 0"
                    :key="'msgdatemarker' + day.dayNum"
                    class="kiwi-messagelist-seperator"
                >
                    <span>{{ (new Date(day.messages[0].time)).toDateString() }}</span>
                </div>

                <template v-for="message in day.messages">
                    <div
                        v-if="shouldShowUnreadMarker(message)"
                        :key="'msgunreadmarker' + message.id"
                        class="kiwi-messagelist-seperator"
                    >
                        <span>{{ $t('unread_messages') }}</span>
                    </div>

                    <div
                        :key="'msg' + message.id"
                        :class="[
                            'kiwi-messagelist-item',
                            selectedMessages[message.id] ?
                                'kiwi-messagelist-item--selected' :
                                ''
                        ]"
                    >
                        <!-- message.template is checked first for a custom component, then each
                            message layout checks for a message.bodyTemplate custom component to
                            apply only to the body area
                        -->
                        <div
                            v-if="message.render() && message.template && message.template.$el"
                            v-rawElement="message.template.$el"
                        />
                        <message-list-message-modern
                            v-else-if="listType === 'modern'"
                            :message="message"
                            :idx="filteredMessages.indexOf(message)"
                            :ml="thisMl"
                        />
                        <message-list-message-inline
                            v-else-if="listType === 'inline'"
                            :message="message"
                            :idx="filteredMessages.indexOf(message)"
                            :ml="thisMl"
                        />
                        <message-list-message-compact
                            v-else-if="listType === 'compact'"
                            :message="message"
                            :idx="filteredMessages.indexOf(message)"
                            :ml="thisMl"
                        />
                    </div>
                </template>
            </div>

            <transition name="kiwi-messagelist-joinloadertrans">
                <div v-if="shouldShowJoiningLoader" class="kiwi-messagelist-joinloader">
                    <LoadingAnimation />
                </div>
            </transition>

            <buffer-key
                v-if="shouldRequestChannelKey"
                :buffer="buffer"
                :network="buffer.getNetwork()"
            />
        </div>
    </div>
</template>

<script>
'kiwi public';

import strftime from 'strftime';
import Logger from '@/libs/Logger';
import * as bufferTools from '@/libs/bufferTools';
import BufferKey from './BufferKey';
import MessageListMessageCompact from './MessageListMessageCompact';
import MessageListMessageModern from './MessageListMessageModern';
import MessageListMessageInline from './MessageListMessageInline';
import LoadingAnimation from './LoadingAnimation.vue';

require('@/libs/polyfill/Element.closest');

let log = Logger.namespace('MessageList.vue');

// If we're scrolled up more than this many pixels, don't auto scroll down to the bottom
// of the message list
const BOTTOM_SCROLL_MARGIN = 60;

export default {
    components: {
        BufferKey,
        MessageListMessageModern,
        MessageListMessageCompact,
        MessageListMessageInline,
        LoadingAnimation,
    },
    props: ['buffer'],
    data() {
        return {
            smooth_scroll: false,
            auto_scroll: true,
            force_smooth_scroll: null,
            chathistoryAvailable: true,
            hover_nick: '',
            message_info_open: null,
            timeToClose: false,
            startClosing: false,
            selectedMessages: Object.create(null),
        };
    },
    computed: {
        thisMl() {
            return this;
        },
        shouldAutoEmbed() {
            if (this.buffer.isChannel() && this.buffer.setting('inline_link_auto_previews')) {
                return true;
            }
            if (this.buffer.isQuery() && this.buffer.setting('inline_link_auto_previews_query')) {
                return true;
            }
            return false;
        },
        listType() {
            if (this.$state.setting('messageLayout')) {
                log.info('Deprecation Warning: The config option \'messageLayout\' has been moved to buffers.messageLayout');
            }
            return this.buffer.setting('messageLayout') || this.$state.setting('messageLayout');
        },
        useExtraFormatting() {
            // Enables simple markdown formatting
            return this.buffer.setting('extra_formatting');
        },
        shouldShowChathistoryTools() {
            // Only show it if we're connected
            if (this.buffer.getNetwork().state !== 'connected') {
                return false;
            }

            let isCorrectBufferType = (this.buffer.isChannel() || this.buffer.isQuery());
            let isSupported = !!this.buffer.getNetwork().ircClient.chathistory.isSupported();
            return isCorrectBufferType && isSupported && this.buffer.flags.chathistory_available;
        },
        shouldRequestChannelKey() {
            return this.buffer.getNetwork().state === 'connected' &&
                this.buffer.isChannel() &&
                this.buffer.flags.channel_badkey;
        },
        ourNick() {
            return this.buffer ?
                this.buffer.getNetwork().nick :
                '';
        },
        filteredMessagesGroupedDay() {
            // Group messages by day
            let days = [];
            let lastDay = null;
            this.filteredMessages.forEach((message) => {
                let day = Math.floor(message.time / 1000 / 86400);
                if (!lastDay || day !== lastDay) {
                    days.push({ dayNum: day, messages: [] });
                    lastDay = day;
                }

                days[days.length - 1].messages.push(message);
            });

            return days;
        },
        filteredMessages() {
            return bufferTools.orderedMessages(this.buffer);
        },
        shouldShowJoiningLoader() {
            return this.buffer.isChannel() &&
                this.buffer.enabled &&
                !this.buffer.joined &&
                this.buffer.getNetwork().state === 'connected';
        },
    },
    watch: {
        filteredMessages() {
            // Data has changed and now preparing to update the DOM.
            // Check our scrolling state before the DOM updates so that we know if we're scrolled
            // at the bottom before new messages are added
            this.checkScrollingState();

            // Wait until after the DOM has updated before possibly scrolling down based on the
            // previous check
            this.$nextTick(() => {
                this.maybeScrollToBottom();
            });
        },
        buffer(newBuffer, oldBuffer) {
            if (oldBuffer) {
                oldBuffer.isMessageTrimming = true;
            }

            if (!newBuffer) {
                return;
            }

            this.message_info_open = null;

            if (this.buffer.getNetwork().state === 'connected') {
                newBuffer.flags.has_opened = true;
            }

            this.auto_scroll = true;
            this.force_smooth_scroll = false;
            this.$nextTick(() => {
                this.scrollToBottom();
            });
        },
    },
    mounted() {
        this.addCopyListeners();

        this.$nextTick(() => {
            this.scrollToBottom();
            // this.smooth_scroll = true;
        });

        this.listen(this.$state, 'mediaviewer.opened', () => {
            this.$nextTick(this.maybeScrollToBottom.apply(this));
        });

        this.listen(this.$state, 'messagelist.scrollto', (opt) => {
            if (opt && opt.id) {
                this.maybeScrollToId(opt.id);
            }
        });
    },
    methods: {
        isHoveringOverMessage(message) {
            return message.nick && message.nick.toLowerCase() === this.hover_nick.toLowerCase();
        },
        toggleMessageInfo(message) {
            if (!message) {
                this.message_info_open = null;
            } else if (this.message_info_open === message) {
                // It's already open, so don't do anything
            } else if (this.canShowInfoForMessage(message)) {
                // If in the process of selecting text, don't show the info box
                let sel = window.getSelection();
                if (sel.rangeCount > 0) {
                    let range = sel.getRangeAt(0);
                    if (range && !range.collapsed) {
                        return;
                    }
                }

                this.message_info_open = message;
                this.$nextTick(this.maybeScrollToBottom.bind(this));
            }
        },
        shouldShowUnreadMarker(message) {
            let idx = this.filteredMessages.indexOf(message);
            let previous = this.filteredMessages[idx - 1];
            let current = this.filteredMessages[idx];
            let lastRead = this.buffer.last_read;

            if (!lastRead) {
                return false;
            }

            if (!current) {
                return false;
            }

            // If the last message has been read, and this message not read
            if (previous && previous.time < lastRead && current.time > lastRead) {
                return true;
            }

            return false;
        },
        shouldShowDateChangeMarker(idx) {
            let previous = this.filteredMessages[idx - 1];
            let current = this.filteredMessages[idx];

            if (!previous && (new Date(current.time)).getDay() !== (new Date()).getDay()) {
                // The first message of the lsit and it's not today
                return true;
            } else if (!previous) {
                // The first message of the lsit but it's today
                return false;
            }

            // If the last message has been read, and this message not read
            if ((new Date(previous.time)).getDay() !== (new Date(current.time)).getDay()) {
                return true;
            }

            return false;
        },
        canShowInfoForMessage(message) {
            let showInfoForTypes = ['privmsg', 'notice', 'action'];
            return showInfoForTypes.indexOf(message.type) > -1;
        },
        bufferSetting(key) {
            return this.buffer.setting(key);
        },
        formatTime(time) {
            return strftime(this.buffer.setting('timestamp_format') || '%T', new Date(time));
        },
        formatTimeFull(time) {
            let format = this.buffer.setting('timestamp_full_format');
            return format ?
                strftime(format, new Date(time)) :
                (new Date(time)).toLocaleString();
        },
        formatMessage(message) {
            return message.toHtml(this);
        },
        isMessageHighlight(message) {
            // Highlighting ourselves when we join or leave a channel is silly
            if (message.type === 'traffic') {
                return false;
            }

            return message.isHighlight;
        },
        userColour(user) {
            if (user && this.bufferSetting('colour_nicknames_in_messages')) {
                return user.getColour();
            }
            return '';
        },
        openUserBox(nick) {
            let user = this.$state.getUser(this.buffer.networkid, nick);
            if (user) {
                this.$state.$emit('userbox.show', user, {
                    buffer: this.buffer,
                });
            }
        },
        onListClick(event) {
            this.toggleMessageInfo();
        },
        onMessageDblClick(event, message) {
            clearTimeout(this.messageClickTmr);

            let dataNick = event.target.getAttribute('data-nick');
            if (!dataNick) {
                return;
            }

            let network = this.buffer.getNetwork();
            let user = network.userByName(dataNick);
            // The user might have left use dataNick as fallback
            let nick = (user && user.nick) ?
                user.nick :
                dataNick;
            this.$state.$emit('input.insertnick', nick);
        },
        onMessageClick(event, message, delay) {
            // Delaying the click for 200ms allows us to check for a second click. ie. double click
            // Quick hack as we only need double click for nicks, nothing else
            if (delay && event.target.getAttribute('data-nick')) {
                clearTimeout(this.messageClickTmr);
                this.messageClickTmr = setTimeout(this.onMessageClick, 200, event, message, false);
                return;
            }

            let isLink = event.target.tagName === 'A';

            let channelName = event.target.getAttribute('data-channel-name');
            if (channelName && isLink) {
                let network = this.buffer.getNetwork();
                this.$state.addBuffer(this.buffer.networkid, channelName);
                network.ircClient.join(channelName);
                this.$state.setActiveBuffer(this.buffer.networkid, channelName);
                return;
            }

            let userNick = event.target.getAttribute('data-nick');
            if (userNick && isLink) {
                this.openUserBox(userNick);
                return;
            }

            let url = event.target.getAttribute('data-url');
            if (url && isLink) {
                if (this.$state.setting('buffers.inline_link_auto_previews')) {
                    message.embed.type = 'url';
                    message.embed.payload = url;
                } else {
                    this.$state.$emit('mediaviewer.show', url);
                }
            }

            if (this.message_info_open && this.message_info_open !== message) {
                // Clicking on another message while another info is open, just close the info
                this.toggleMessageInfo();
                event.preventDefault();
                return;
            }

            if (this.$state.ui.is_touch && this.$state.setting('buffers.show_message_info')) {
                if (this.canShowInfoForMessage(message) && event.target.nodeName === 'A') {
                    // We show message info boxes on touch screen devices so that the user has an
                    // option to preview the links or do other stuff.
                    event.preventDefault();
                }

                this.toggleMessageInfo(message);
            }
        },
        checkScrollingState() {
            let el = this.$el;
            let scrolledUpByPx = el.scrollHeight - (el.offsetHeight + el.scrollTop);

            // We need to know at this point (before the DOM has updated with new messages) if we
            // are at the bottom of the messagelist or not, otherwise once the DOM has updated then
            // it is too late to determine if we should auto scroll down
            if (scrolledUpByPx > BOTTOM_SCROLL_MARGIN) {
                this.auto_scroll = false;
                this.buffer.isMessageTrimming = false;
            } else {
                this.auto_scroll = true;
                this.buffer.isMessageTrimming = true;
            }

            if (this.force_smooth_scroll !== null) {
                this.smooth_scroll = this.force_smooth_scroll;
                this.force_smooth_scroll = null;
            // TODO: Enabling smooth_scroll breaks the auto-scroll-to-bottom on fast buffers as
            //       it takes time to scroll down and it looks like we're scrolled too far up when
            //       detecting if were scrolled up or not. Look into ways around this so that we
            //       can enable it as it does look a lot better.
            // } else if (scrolledUpByPx < BOTTOM_SCROLL_MARGIN) {
            //    this.smooth_scroll = true;
            } else {
                this.smooth_scroll = false;
            }
        },
        onListResize(e) {
            // The messagelist or interface has resized or had new content added
            // check if we should auto scroll down to the bottom
            this.maybeScrollToBottom();
        },
        scrollToBottom() {
            this.$el.scrollTop = this.$el.scrollHeight;
        },
        maybeScrollToBottom() {
            if (this.auto_scroll) {
                this.scrollToBottom();
            }
        },
        maybeScrollToId(id) {
            let messageElement = this.$el.querySelector('.kiwi-messagelist-message[data-message-id="' + id + '"]');
            if (messageElement && messageElement.offsetTop) {
                this.$el.scrollTop = messageElement.offsetTop;
                this.auto_scroll = false;
            }
        },
        getSelectedMessages() {
            let sel = document.getSelection();
            let r = sel.getRangeAt(0);
            let messageEls = [...this.$el.querySelectorAll('.kiwi-messagelist-message')];
            let selectedMessageEls = messageEls.filter((el) => r.intersectsNode(el));
            return selectedMessageEls;
        },
        restrictTextSelection() { // Prevents the selection cursor escaping the message list.
            document.querySelector('body').classList.add('kiwi-unselectable');
            this.$el.style.userSelect = 'text';
        },
        unrestrictTextSelection() { // Allows all page elements to be selected again.
            document.querySelector('body').classList.remove('kiwi-unselectable');
            this.$el.style.userSelect = 'auto';
        },
        removeSelections(removeNative = false) {
            this.selectedMessages = Object.create(null);

            let selection = document.getSelection();
            if (removeNative && selection) {
                // stops the native browser selection being left behind after ctrl+c
                selection.removeAllRanges();
            }
        },
        addCopyListeners() { // Better copy pasting
            const LogFormatter = (msg) => {
                let text = '';

                switch (msg.type) {
                case 'privmsg':
                    text = `<${msg.nick}> ${msg.message}`;
                    break;
                case 'nick':
                case 'mode':
                case 'action':
                case 'traffic':
                    text = `${msg.message}`;
                    break;
                default:
                    text = msg.message;
                }
                if (text.length) {
                    return `[${(new Date(msg.time)).toLocaleTimeString({ hour: '2-digit', minute: '2-digit', second: '2-digit' })}] ${text}`;
                }
                return null;
            };

            let copyData = '';
            let selecting = false;
            let selectionChangeOff = null;

            this.listen(document, 'selectstart', (e) => {
                if (!this.$el.contains(e.target)) {
                    // Selected elsewhere on the page
                    copyData = '';
                    this.removeSelections();
                    return;
                }

                this.removeSelections();
                selectionChangeOff = this.listen(document, 'selectionchange', onSelectionChange);
            });

            this.listen(document, 'mouseup', (e) => {
                selectionChangeOff && selectionChangeOff();
                this.unrestrictTextSelection();
                if (selecting) {
                    e.preventDefault();
                }
                selecting = false;
            });

            let onSelectionChange = (e) => {
                if (!this.$el) {
                    return true;
                }

                copyData = ''; // Store the text data to be copied in this.

                let selection = document.getSelection();

                if (!selection
                || !selection.anchorNode
                || !selection.anchorNode.parentNode.closest('.' + this.$el.className)) {
                    this.unrestrictTextSelection();
                    this.removeSelections();
                    return true;
                }

                this.removeSelections();
                // Prevent the selection escaping the message list
                this.restrictTextSelection();
                if (selection.rangeCount > 0) {
                    selecting = true;

                    let selectedMesssageEls = this.getSelectedMessages();
                    let selectedMessages = [];
                    selectedMesssageEls.forEach((el) => {
                        let m = this.buffer.messagesObj.messageIds[el.dataset.messageId];
                        if (m) {
                            selectedMessages.push(m);
                        }
                    });

                    // If only 1 message is selected then treat the selection as native text
                    // selection. Most likely copying part of a message only.
                    if (selectedMessages.length === 1) {
                        selectedMessages = [];
                    }

                    this.selectedMessages = Object.create(null);
                    selectedMessages.forEach((m) => {
                        this.selectedMessages[m.id] = m;
                    });

                    // Iterate through the selected messages, format and store as a
                    // string to be used in the copy handler
                    copyData = selectedMessages
                        .sort((a, b) => (a.time > b.time ? 1 : -1))
                        .filter((m) => m.message.trim().length)
                        .map(LogFormatter)
                        .join('\r\n');
                } else {
                    this.unrestrictTextSelection();
                }
                return false;
            };

            this.listen(document, 'copy', (e) => {
                if (!copyData || !copyData.length) { // Just do a normal copy if no special data
                    return true;
                }

                if (navigator.clipboard) { // Supports Clipboard API
                    navigator.clipboard.writeText(copyData);
                } else {
                    let input = document.createElement('textarea');
                    document.body.appendChild(input);
                    input.innerHTML = copyData;
                    input.select();
                    document.execCommand('copy');
                    document.body.removeChild(input);
                }

                return true;
            });
        },
        // Move a messages embeded content to the main media preview
        openEmbedInPreview(message) {
            // First open the embed in the main media preview
            let embed = message.embed;
            if (embed.type === 'url') {
                this.$state.$emit('mediaviewer.show', embed.payload);
            } else if (embed.type === 'component') {
                this.$state.$emit('mediaviewer.show', {
                    component: embed.payload,
                });
            }

            // Remove the embed from the message
            embed.payload = null;
        },
    },
};
</script>

<style lang="less">

.kiwi-unselectable * {
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
}

div.kiwi-messagelist-item.kiwi-messagelist-item--selected {
    border-left: 7px solid var(--brand-primary);
    transform: translateX(20px);
    transition: transform 0.1s;
}

div.kiwi-messagelist-item.kiwi-messagelist-item--selected .kiwi-messagelist-message {
    border-left-width: 0;
}

.kiwi-messagelist-item.kiwi-messagelist-item--selected .kiwi-messagelist-message *::selection {
    background-color: unset;
    color: unset;
}

.kiwi-unselectable .kiwi-messagelist-scrollback {
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
}

.kiwi-messagelist {
    overflow-y: auto;
    overflow-x: hidden;
    box-sizing: border-box;
    margin-bottom: 25px;
    position: relative;
}

.kiwi-messagelist--smoothscroll {
    scroll-behavior: smooth;
}

.kiwi-messagelist * {
    user-select: text;
}

.kiwi-messagelist::-webkit-scrollbar-track {
    border-radius: 10px;
    background: transparent;
}

.kiwi-messagelist::-webkit-scrollbar {
    width: 8px;
    background: transparent;
}

.kiwi-messagelist::-webkit-scrollbar-thumb {
    border-radius: 3px;
}

.kiwi-messagelist-item {
    /* Allow child elements to make use of margins+padding within messagelist items */
    overflow: hidden;
}

.kiwi-messagelist-message {
    padding: 0 10px;

    /* some message highlights add a left border so add a default invisble one in preperation */
    border-left: 3px solid transparent;
    overflow: hidden;
    line-height: 1.5em;
    margin: 0;
}

.kiwi-wrap--monospace .kiwi-messagelist-message,
.kiwi-messagelist-message.kiwi-messagelist-message-help {
    font-family: Consolas, monaco, monospace;
    font-size: 80%;
}

/* Remove the styling for none user messages, as they make the page look bloated */
.kiwi-messagelist-message-mode,
.kiwi-messagelist-message-traffic {
    padding-top: 5px;
    padding-bottom: 5px;
    min-height: 0;
    line-height: normal;
    text-align: left;
}

/* Remove the min height from the message, as again, makes the page look bloated */
.kiwi-messagelist-body {
    min-height: 0;
    text-align: left;
    line-height: 1.5em;
    font-size: 1.05em;
    margin: 0;
    padding: 0;
}

/* Channel messages - e.g 'server on #testing22 ' message and such */
.kiwi-messagelist-message-mode,
.kiwi-messagelist-message-traffic,
.kiwi-messagelist-message-nick {
    margin: 10px 0;
    opacity: 0.85;
    text-align: center;
    border: none;

    &:hover {
        opacity: 1;
    }
}

/* Absolute position the time on these messages so it's not above the message, it looks awful */
.kiwi-messagelist-message-mode .kiwi-messagelist-time,
.kiwi-messagelist-message-traffic .kiwi-messagelist-time {
    position: absolute;
    top: 1px;
    right: 10px;
}

.kiwi-messagelist-message--authorrepeat {
    border-top: none;
}

.kiwi-messagelist-message--authorrepeat .kiwi-messagelist-nick,
.kiwi-messagelist-message--authorrepeat .kiwi-messagelist-time {
    /* Setting the opacity instead visible:none ensures it's still selectable when copying text */
    opacity: 0;
    cursor: default;
}

.kiwi-container--sidebar-drawn .kiwi-messagelist::after {
    content: '';
    z-index: 3;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    opacity: 0.5;
    position: fixed;
    pointer-events: none;
}

.kiwi-container--sidebar-drawn.kiwi-container--no-sidebar .kiwi-messagelist::after {
    width: 0;
    height: 0;
    display: none;
    pointer-events: inherit;
    position: static;
    z-index: 0;
}

.kiwi-messagelist-scrollback {
    text-align: center;
    padding: 5px;
}

.kiwi-messagelist-seperator + .kiwi-messagelist-message {
    border-top: none;
}

.kiwi-messagelist-message--blur {
    opacity: 0.3;
}

.kiwi-messagelist-nick {
    text-align: right;
    font-weight: bold;
    text-overflow: ellipsis;
    overflow: hidden;
    vertical-align: top;
    cursor: pointer;
    padding: 2px 4px;
    word-break: break-all;
}

.kiwi-messagelist-message-traffic .kiwi-messagelist-nick {
    display: none;
}

.kiwi-messagelist-seperator {
    text-align: center;
    display: block;
    margin: 1em auto;
    position: sticky;
    top: -1px;
    z-index: 1;
}

.kiwi-messagelist-seperator > span {
    display: inline-block;
    position: relative;
    z-index: 1;
    padding: 0 1em;
    user-select: none;
}

/** Displaying an emoji in a message */
.kiwi-messagelist-emoji {
    width: 1.3em;
    display: inline-block;
    vertical-align: middle;
}

@keyframes emojiIn {
    0% {
        transform: scale(0);
    }

    100% {
        transform: scale(1);
    }
}

.kiwi-messagelist-emoji--single {
    animation: 0.1s ease-in-out 0s 1 emojiIn;
    font-size: 2em;
}

/** Message structure */
.kiwi-messagelist-body .kiwi-nick {
    cursor: pointer;
}

.kiwi-messagelist-nick:hover {
    overflow: visible;
    width: auto;
}

/* Topic changes */
.kiwi-messagelist-message-topic {
    border-radius: 5px;
    margin: 18px;
    margin-left: 0;
    padding: 5px;
    text-align: center;
    position: relative;
    min-height: 0;
    display: block;
}

.kiwi-messagelist-message-topic .kiwi-messagelist-body {
    min-height: 0;
    margin: 0;

    &::before {
        display: none;
    }
}

.kiwi-messagelist-message-topic.kiwi-messagelist-message-topic .kiwi-messagelist-time {
    display: none;
}

.kiwi-messagelist-message-topic.kiwi-messagelist-message-topic .kiwi-messagelist-nick {
    display: none;
}

/* Actions */
.kiwi-messagelist-message-action .kiwi-messagelist-message-body {
    font-style: italic;
}

/* Traffic (joins, parts, quits, kicks) */
.kiwi-messagelist-message-traffic.kiwi-messagelist-message-traffic .kiwi-messagelist-nick {
    display: none;
}

.kiwi-messagelist-message-traffic .kiwi-messagelist-body {
    font-style: italic;
}

.kiwi-messagelist-message-action.kiwi-messagelist-message-action .kiwi-messagelist-nick {
    display: none;
}

/* MOTD */
.kiwi-messagelist-message-motd {
    font-family: monospace;
}

.kiwi-messagelist-message.kiwi-messagelist-message--hover,
.kiwi-messagelist-message.kiwi-messagelist-message--highlight,
.kiwi-messagelist-message.kiwi-messagelist-message-traffic--hover {
    position: relative;
}

/* Links */
.kiwi-messagelist-message-linkhandle {
    margin-left: 4px;
    font-size: 0.8em;
}

.kiwi-wrap--touch .kiwi-messagelist-message-linkhandle {
    display: none;
}

.kiwi-messagelist-joinloader {
    margin: 1em auto;
    width: 100px;

    /* the magic number below is the exact ratio of the kiwi logo height/width */
    height: calc (100px * 0.85987261146496815286624203821656);
    overflow: hidden;
}

.kiwi-messagelist-joinloadertrans-enter,
.kiwi-messagelist-joinloadertrans-leave-to {
    height: 0;
    opacity: 0;
}

.kiwi-messagelist-joinloadertrans-enter-to,
.kiwi-messagelist-joinloadertrans-leave {
    height: 150px;
    opacity: 1;
}

.kiwi-messagelist-joinloadertrans-enter-active,
.kiwi-messagelist-joinloadertrans-leave-active {
    transition: height 0.5s, opacity 0.5s;
}

@media screen and (max-width: 700px) {
    .kiwi-messagelist-message,
    .kiwi-messageinfo {
        margin: 0;
    }
}

</style>
