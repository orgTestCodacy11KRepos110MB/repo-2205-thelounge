<template>
	<div
		ref="chat"
		class="chat"
		tabindex="-1">
		<div :class="['show-more', { show: channel.moreHistoryAvailable }]">
			<button
				ref="loadMoreButton"
				:disabled="channel.historyLoading || !$root.isConnected"
				class="btn"
				@click="onShowMoreClick">
				<span v-if="channel.historyLoading">Loading…</span>
				<span v-else>Show older messages</span>
			</button>
		</div>
		<div
			class="messages"
			role="log"
			aria-live="polite"
			aria-relevant="additions"
			@copy="onCopy">
			<template v-for="(message, id) in condensedMessages">
				<DateMarker
					v-if="shouldDisplayDateMarker(message, id)"
					:key="message.id + '-date'"
					:message="message" />
				<div
					v-if="shouldDisplayUnreadMarker(message.id)"
					:key="message.id + '-unread'"
					class="unread-marker">
					<span class="unread-marker-text" />
				</div>

				<MessageCondensed
					v-if="message.type === 'condensed'"
					:key="message.id"
					:network="network"
					:keep-scroll-position="keepScrollPosition"
					:messages="message.messages" />
				<Message
					v-else
					:key="message.id"
					:network="network"
					:message="message"
					:keep-scroll-position="keepScrollPosition"
					@linkPreviewToggle="onLinkPreviewToggle" />
			</template>
		</div>
	</div>
</template>

<script>
require("intersection-observer");

const constants = require("../js/constants");
const clipboard = require("../js/clipboard");
import socket from "../js/socket";
import Message from "./Message.vue";
import MessageCondensed from "./MessageCondensed.vue";
import DateMarker from "./DateMarker.vue";

export default {
	name: "MessageList",
	components: {
		Message,
		MessageCondensed,
		DateMarker,
	},
	props: {
		network: Object,
		channel: Object,
	},
	computed: {
		condensedMessages() {
			if (this.channel.type !== "channel") {
				return this.channel.messages;
			}

			// If actions are hidden, just return a message list with them excluded
			if (this.$root.settings.statusMessages === "hidden") {
				return this.channel.messages.filter((message) => !constants.condensedTypes.includes(message.type));
			}

			// If actions are not condensed, just return raw message list
			if (this.$root.settings.statusMessages !== "condensed") {
				return this.channel.messages;
			}

			const condensed = [];
			let lastCondensedContainer = null;

			for (const message of this.channel.messages) {
				// If this message is not condensable, or its an action affecting our user,
				// then just append the message to container and be done with it
				if (message.self || message.highlight || !constants.condensedTypes.includes(message.type)) {
					lastCondensedContainer = null;

					condensed.push(message);

					continue;
				}

				if (lastCondensedContainer === null) {
					lastCondensedContainer = {
						time: message.time,
						type: "condensed",
						messages: [],
					};

					condensed.push(lastCondensedContainer);
				}

				lastCondensedContainer.messages.push(message);

				// Set id of the condensed container to last message id,
				// which is required for the unread marker to work correctly
				lastCondensedContainer.id = message.id;

				// If this message is the unread boundary, create a split condensed container
				if (message.id === this.channel.firstUnread) {
					lastCondensedContainer = null;
				}
			}

			return condensed;
		},
	},
	watch: {
		"channel.id"() {
			this.channel.scrolledToBottom = true;

			// Re-add the intersection observer to trigger the check again on channel switch
			// Otherwise if last channel had the button visible, switching to a new channel won't trigger the history
			if (this.historyObserver) {
				this.historyObserver.unobserve(this.$refs.loadMoreButton);
				this.historyObserver.observe(this.$refs.loadMoreButton);
			}
		},
		"channel.messages"() {
			this.keepScrollPosition();
		},
		"channel.pendingMessage"() {
			this.$nextTick(() => {
				// Keep the scroll stuck when input gets resized while typing
				this.keepScrollPosition();
			});
		},
	},
	created() {
		this.$nextTick(() => {
			if (!this.$refs.chat) {
				return;
			}

			if (window.IntersectionObserver) {
				this.historyObserver = new window.IntersectionObserver(this.onLoadButtonObserved, {
					root: this.$refs.chat,
				});
			}

			this.jumpToBottom();
		});
	},
	mounted() {
		this.$refs.chat.addEventListener("scroll", this.handleScroll, {passive: true});

		this.$root.$on("resize", this.handleResize);

		this.$nextTick(() => {
			if (this.historyObserver) {
				this.historyObserver.observe(this.$refs.loadMoreButton);
			}
		});
	},
	beforeUpdate() {
		this.unreadMarkerShown = false;
	},
	beforeDestroy() {
		this.$root.$off("resize", this.handleResize);
		this.$refs.chat.removeEventListener("scroll", this.handleScroll);
	},
	destroyed() {
		if (this.historyObserver) {
			this.historyObserver.disconnect();
		}
	},
	methods: {
		shouldDisplayDateMarker(message, id) {
			const previousMessage = this.condensedMessages[id - 1];

			if (!previousMessage) {
				return true;
			}

			return (new Date(previousMessage.time)).getDay() !== (new Date(message.time)).getDay();
		},
		shouldDisplayUnreadMarker(id) {
			if (!this.unreadMarkerShown && id > this.channel.firstUnread) {
				this.unreadMarkerShown = true;
				return true;
			}

			return false;
		},
		onCopy() {
			clipboard(this.$el);
		},
		onLinkPreviewToggle(preview, message) {
			this.keepScrollPosition();

			// Tell the server we're toggling so it remembers at page reload
			// TODO Avoid sending many single events when using `/collapse` or `/expand`
			// See https://github.com/thelounge/thelounge/issues/1377
			socket.emit("msg:preview:toggle", {
				target: this.channel.id,
				msgId: message.id,
				link: preview.link,
				shown: preview.shown,
			});
		},
		onShowMoreClick() {
			let lastMessage = this.channel.messages[0];
			lastMessage = lastMessage ? lastMessage.id : -1;

			this.channel.historyLoading = true;

			socket.emit("more", {
				target: this.channel.id,
				lastId: lastMessage,
			});
		},
		onLoadButtonObserved(entries) {
			entries.forEach((entry) => {
				if (!entry.isIntersecting) {
					return;
				}

				this.onShowMoreClick();
			});
		},
		keepScrollPosition() {
			// If we are already waiting for the next tick to force scroll position,
			// we have no reason to perform more checks and set it again in the next tick
			if (this.isWaitingForNextTick) {
				return;
			}

			const el = this.$refs.chat;

			if (!el) {
				return;
			}

			if (!this.channel.scrolledToBottom) {
				if (this.channel.historyLoading) {
					const heightOld = el.scrollHeight - el.scrollTop;

					this.isWaitingForNextTick = true;
					this.$nextTick(() => {
						this.isWaitingForNextTick = false;
						this.skipNextScrollEvent = true;
						el.scrollTop = el.scrollHeight - heightOld;
					});
				}

				return;
			}

			this.isWaitingForNextTick = true;
			this.$nextTick(() => {
				this.isWaitingForNextTick = false;
				this.jumpToBottom();
			});
		},
		handleScroll() {
			// Setting scrollTop also triggers scroll event
			// We don't want to perform calculations for that
			if (this.skipNextScrollEvent) {
				this.skipNextScrollEvent = false;
				return;
			}

			const el = this.$refs.chat;

			if (!el) {
				return;
			}

			this.channel.scrolledToBottom = el.scrollHeight - el.scrollTop - el.offsetHeight <= 30;
		},
		handleResize() {
			// Keep message list scrolled to bottom on resize
			if (this.channel.scrolledToBottom) {
				this.jumpToBottom();
			}
		},
		jumpToBottom() {
			this.skipNextScrollEvent = true;
			this.channel.scrolledToBottom = true;

			const el = this.$refs.chat;
			el.scrollTop = el.scrollHeight;
		},
	},
};
</script>