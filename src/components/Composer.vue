<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div class="new-post" data-id="">
		<div class="new-post-author">
			<avatar :user="currentUser.uid" :display-name="currentUser.displayName" :disable-tooltip="true"
				:size="32" />
			<div class="post-author">
				<span class="post-author-name">
					{{ currentUser.displayName }}
				</span>
				<span class="post-author-id">
					{{ socialId }}
				</span>
			</div>
		</div>
		<div v-if="replyTo" class="reply-to">
			<p>
				<span>In reply to</span>
				<actor-avatar :actor="replyTo.actor_info" :size="16" />
				<strong>{{ replyTo.actor_info.account }}</strong>
				<a class="icon-close" @click="replyTo=null" />
			</p>
			<div class="reply-to-preview">
				{{ replyTo.content }}
			</div>
		</div>
		<form class="new-post-form" @submit.prevent="createPost">
			<vue-tribute :options="tributeOptions">
				<!-- eslint-disable-next-line vue/valid-v-model -->
				<div ref="composerInput" v-contenteditable:post.dangerousHTML="canType && !loading" class="message"
					placeholder="What would you like to share?" :class="{'icon-loading': loading}" @keyup.prevent.enter="keyup"
					@tribute-replaced="updatePostFromTribute" />
			</vue-tribute>
			<emoji-picker ref="emojiPicker" :search="search" class="emoji-picker-wrapper"
				@emoji="insert">
				<div slot="emoji-invoker" v-tooltip="'Insert emoji'" slot-scope="{ events }"
					class="emoji-invoker" tabindex="0" @keyup.enter="events.click"
					@keyup.space="events.click" @click.stop="events.click" />
				<!-- eslint-disable-next-line vue/no-template-shadow -->
				<div slot="emoji-picker" slot-scope="{ emojis, insert }" class="emoji-picker popovermenu">
					<div>
						<div>
							<input v-model="search" v-focus-on-create type="text"
								@keyup.enter="insert(emojis)">
						</div>
						<div>
							<div v-for="(emojiGroup, category) in emojis" :key="category">
								<h5>{{ category }}</h5>
								<div>
									<!-- eslint-disable vue/no-v-html -->
									<span v-for="(emoji, emojiName) in emojiGroup" :key="emojiName" :title="emojiName"
										tabindex="0"
										class="emoji" @click="insert(emoji)" @keyup.enter="insert(emoji)"
										@keyup.space="insert(emoji)" v-html="$twemoji.parse(emoji)" />
								</div>
							</div>
						</div>
					</div>
				</div>
			</emoji-picker>

			<masonry>
				<div v-for="(item, index) in miniatures" :key="index" ref="miniatures">
					<img :src="item.img" :usemap="'#map' + index">
					<map :name="'map' + index">
						<area shape="circle" :coords="getImageMapCoords(index)" @click="removeAttachment(index)">
					</map>
				</div>
			</masonry>

			<div class="options">
				<input :value="currentVisibilityPostLabel" :disabled="post.length < 1 || post==='<br>'" class="submit primary"
					type="submit" title="" data-original-title="Post">
				<div v-click-outside="hidePopoverMenu">
					<button :class="currentVisibilityIconClass" @click.prevent="togglePopoverMenu" />
					<div :class="{open: menuOpened}" class="popovermenu menu-center">
						<popover-menu :menu="visibilityPopover" />
					</div>
				</div>
				<div class="emptySpace" />
				<label v-tooltip="'Attach document'" class="button icon-upload" for="file-upload" />
				<input id="file-upload" ref="addAttach" class="upload-button"
					type="file" @change="AddAttachment">
			</div>
		</form>
	</div>
</template>
<style scoped lang="scss">

	.new-post {
		padding: 10px;
		background-color: var(--color-main-background);
		position: sticky;
		top: 47px;
		z-index: 100;
		margin-bottom: 10px;
	}

	.new-post-author {
		padding: 5px;
		display: flex;
		flex-wrap: wrap;

		.post-author {
			padding: 6px;

			.post-author-name {
				font-weight: bold;
			}

			.post-author-id {
				opacity: .7;
			}
		}
	}

	.reply-to {
		background-image: url(../../img/reply.svg);
		background-position: 5px 5px;
		background-repeat: no-repeat;
		margin-left: 39px;
		margin-bottom: 20px;
		overflow: hidden;
		background-color: #fafafa;
		border-radius: 3px;
		padding: 5px;
		padding-left: 30px;

		.icon-close {
			display: inline-block;
			float: right;
			opacity: .7;
			padding: 3px;
		}
	}

	.new-post-form {
		flex-grow: 1;
		position: relative;
		top: -10px;
		margin-left: 39px;
	}

	.message {
		width: 100%;
		padding-right: 44px;
		min-height: 70px;
		min-width: 2px;
		display: block;
	}

	[contenteditable=true]:empty:before {
		content: attr(placeholder);
		display: block; /* For Firefox */
		opacity: .5;
	}

	input[type=submit].inline {
		width: 44px;
		height: 44px;
		margin: 0;
		padding: 13px;
		background-color: transparent;
		border: none;
		opacity: 0.3;
		position: absolute;
		bottom: 0;
		right: 0;
	}

	.options {
		display: flex;
		align-items: flex-end;
		width: 100%;
		flex-direction: row-reverse;
	}

	.options > div {
		position: relative;
	}

	.options button {
		width: 34px;
		height: 34px;
	}

	.emptySpace {
		flex-grow:1;
	}

	.icon-upload {
		width: 34px;
		height: 34px;
	}

	.upload-button {
		display: none;
	}

	.emoji-invoker {
		background-image: var(--icon-social-emoji-000);
		background-position: center center;
		background-repeat: no-repeat;
		width: 38px;
		opacity: 0.5;
		background-size: 16px 16px;
		height: 38px;
		cursor: pointer;
		display: block;
	}

	.emoji-invoker:focus,
	.emoji-invoker:hover {
		opacity: 1;
	}

	.emoji-picker-wrapper {
		position: absolute;
		right: 0;
		top: 0;
	}

	.emoji-picker.popovermenu {
		display: block;
		padding: 5px;
		width: 200px;
		height: 200px;
		top: 44px;
	}

	.emoji-picker > div {
		overflow: hidden;
		overflow-y: scroll;
		height: 190px;
	}

	.emoji-picker input {
		width: 100%;
	}

	.emoji-picker span.emoji {
		padding: 3px;
	}

	.emoji-picker span.emoji:focus {
		background-color: var(--color-background-dark);
	}

	.emoji-picker .emoji img {
		width: 16px;
	}

	.popovermenu {
		top: 55px;
	}

	.attachment-picker-wrapper {
		position: absolute;
		right: 0;
		top: 2;
	}
</style>
<style>
	/* Tribute-specific styles TODO: properly scope component css */
	.tribute-container {
		position: absolute;
		top: 0;
		left: 0;
		height: auto;
		max-height: 300px;
		max-width: 500px;
		min-width: 200px;
		overflow: auto;
		display: block;
		z-index: 999999;
		border-radius: 4px;
		box-shadow: 0 1px 3px var(--color-box-shadow);
	}

	.tribute-container ul {
		margin: 0;
		margin-top: 2px;
		padding: 0;
		list-style: none;
		background: var(--color-main-background);
		border-radius: 4px;
		background-clip: padding-box;
		overflow: hidden;
	}

	.tribute-container li {
		color: var(--color-text);
		padding: 5px 10px;
		cursor: pointer;
		font-size: 14px;
		display: flex;
	}

	.tribute-container li span {
		display: block;
	}

	.tribute-container li.highlight,
	.tribute-container li:hover {
		background: var(--color-primary);
		color: var(--color-primary-text);
	}

	.tribute-container li img {
		width: 32px;
		height: 32px;
		border-radius: 50%;
		overflow: hidden;
		margin-right: 10px;
		margin-left: -3px;
		margin-top: 3px;
	}

	.tribute-container li span {
		font-weight: bold;
	}

	.tribute-container li.no-match {
		cursor: default;
	}

	.tribute-container .menu-highlighted {
		font-weight: bold;
	}

	.tribute-container .account,
	.tribute-container li.highlight .account,
	.tribute-container li:hover .account {
		font-weight: normal;
		color: var(--color-text-light);
		opacity: 0.5;
	}

	.tribute-container li.highlight .account,
	.tribute-container li:hover .account {
		color: var(--color-primary-text) !important;
		opacity: .6;
	}

	.message .mention {
		color: var(--color-primary-element);
		background-color: var(--color-background-dark);
		border-radius: 5px;
		padding-top: 1px;
		padding-left: 2px;
		padding-bottom: 1px;
		padding-right: 5px;
	}

	.mention img {
		width: 16px;
		border-radius: 50%;
		overflow: hidden;
		margin-right: 3px;
		vertical-align: middle;
		margin-top: -1px;
	}

	.hashtag {
		text-decoration: underline;
	}
</style>
<script>

import Avatar from 'nextcloud-vue/dist/Components/Avatar'
import PopoverMenu from 'nextcloud-vue/dist/Components/PopoverMenu'
import EmojiPicker from 'vue-emoji-picker'
import VueTribute from 'vue-tribute'
import he from 'he'
import CurrentUserMixin from './../mixins/currentUserMixin'
import FocusOnCreate from '../directives/focusOnCreate'
import axios from 'nextcloud-axios'
import ActorAvatar from './ActorAvatar.vue'

export default {
	name: 'Composer',
	components: {
		PopoverMenu,
		Avatar,
		ActorAvatar,
		EmojiPicker,
		VueTribute
	},
	directives: {
		FocusOnCreate: FocusOnCreate
	},
	mixins: [CurrentUserMixin],
	props: {},
	data() {
		return {
			type: localStorage.getItem('social.lastPostType') || 'followers',
			loading: false,
			post: '',
			miniatures: [],		// miniatures of images stored in postAttachments
			postAttachments: [],	// The toot's attachments
			canType: true,
			search: '',
			replyTo: null,
			tributeOptions: {
				spaceSelectsMatch: true,
				collection: [
					{
						trigger: '@',
						lookup: function(item) {
							return item.key + item.value
						},
						menuItemTemplate: function(item) {
							return '<img src="' + item.original.avatar + '" /><div>'
								+ '<span class="displayName">' + item.original.key + '</span>'
								+ '<span class="account">' + item.original.value + '</span>'
								+ '</div>'
						},
						selectTemplate: function(item) {
							return '<span class="mention" contenteditable="false">'
								+ '<a href="' + item.original.url + '" target="_blank"><img src="' + item.original.avatar + '" />@' + item.original.value + '</a></span>'
						},
						values: (text, cb) => {
							let users = []

							if (text.length < 1) {
								cb(users)
							}
							this.remoteSearchAccounts(text).then((result) => {
								for (var i in result.data.result.accounts) {
									let user = result.data.result.accounts[i]
									users.push({
										key: user.preferredUsername,
										value: user.account,
										url: user.url,
										avatar: user.local ? OC.generateUrl(`/avatar/${user.preferredUsername}/32`) : OC.generateUrl(`apps/social/api/v1/global/actor/avatar?id=${user.id}`)
									})
								}
								cb(users)
							})
						}
					},
					{
						trigger: '#',
						menuItemTemplate: function(item) {
							return item.original.value
						},
						selectTemplate: function(item) {
							let tag = ''
							// item is undefined if selectTemplate is called from a noMatchTemplate menu
							if (typeof item === 'undefined') {
								tag = this.currentMentionTextSnapshot
							} else {
								tag = item.original.value
							}
							return '<span class="hashtag" contenteditable="false">'
								+ '<a href="' + OC.generateUrl('/timeline/tags/' + tag) + '" target="_blank">#' + tag + '</a></span>'
						},
						values: (text, cb) => {
							let tags = []

							if (text.length < 1) {
								cb(tags)
							}
							this.remoteSearchHashtags(text).then((result) => {
								if (result.data.result.exact) {
									tags.push({
										key: result.data.result.exact,
										value: result.data.result.exact
									})
								}
								for (var i in result.data.result.tags) {
									let tag = result.data.result.tags[i]
									tags.push({
										key: tag.hashtag,
										value: tag.hashtag
									})
								}
								cb(tags)
							})
						}
					}
				],
				noMatchTemplate() {
					if (this.current.collection.trigger === '#') {
						if (this.current.mentionText === '') {
							return undefined
						} else {
							return '<li data-index="0">#' + this.current.mentionText + '</li>'
						}
					}
				}
			},
			menuOpened: false

		}
	},
	computed: {
		currentVisibilityIconClass() {
			return this.visibilityIconClass(this.type)
		},
		visibilityIconClass() {
			return (type) => {
				if (typeof type === 'undefined') {
					type = this.type
				}
				switch (type) {
				case 'public':
					return 'icon-link'
				case 'followers':
					return 'icon-contacts-dark'
				case 'direct':
					return 'icon-external'
				case 'unlisted':
					return 'icon-password'
				}
			}
		},
		currentVisibilityPostLabel() {
			return this.visibilityPostLabel(this.type)
		},
		visibilityPostLabel() {
			return (type) => {
				if (typeof type === 'undefined') {
					type = this.type
				}
				switch (type) {
				case 'public':
					return t('social', 'Post publicly')
				case 'followers':
					return t('social', 'Post to followers')
				case 'direct':
					return t('social', 'Post to recipients')
				case 'unlisted':
					return t('social', 'Post unlisted')
				}
			}
		},
		activeState() {
			return (type) => {
				if (type === this.type) {
					return true
				} else {
					return false
				}
			}
		},
		visibilityPopover() {
			return [
				{
					action: () => {
						this.switchType('public')
					},
					icon: this.visibilityIconClass('public'),
					active: this.activeState('public'),
					text: t('social', 'Public'),
					longtext: t('social', 'Post to public timelines')
				},
				{
					action: () => {
						this.switchType('unlisted')
					},
					icon: this.visibilityIconClass('unlisted'),
					active: this.activeState('unlisted'),
					text: t('social', 'Unlisted'),
					longtext: t('social', 'Do not post to public timelines')
				},
				{
					action: () => {
						this.switchType('followers')
					},
					icon: this.visibilityIconClass('followers'),
					active: this.activeState('followers'),
					text: t('social', 'Followers'),
					longtext: t('social', 'Post to followers only')
				},
				{
					action: () => {
						this.switchType('direct')
					},
					icon: this.visibilityIconClass('direct'),
					active: this.activeState('direct'),
					text: t('social', 'Direct'),
					longtext: t('social', 'Post to mentioned users only')
				}
			]
		}
	},
	mounted() {
		this.$root.$on('composer-reply', (data) => {
			this.replyTo = data
		})
	},
	methods: {
		AddAttachment() {
			// TODO: handle (or prevent) mulitple files
			let self = this
			let file = this.$refs.addAttach.files[0]
			let reader = new FileReader()

			// Called when selected file is completly loaded to draw a miniature
			reader.onload = function(e) {
				let canvas = document.createElement('canvas')
				let ctx = canvas.getContext('2d')
				let width = 265
				let height = 180
				let img = new Image()

				// Called when img.src is set below
				img.onload = function() {

					// scale image for miniature
					let imgWidth = this.width
					let imgHeight = this.height
					imgHeight = Math.floor(imgHeight * (width / imgWidth))
					imgWidth = width
					if (imgHeight > height) {
						imgWidth = Math.floor(imgWidth * (height / imgHeight))
						imgHeight = height
					}
					canvas.width = imgWidth
					canvas.height = imgHeight
					ctx.drawImage(this, 0, 0, imgWidth, imgHeight)

					// Draw a border
					ctx.beginPath()
					ctx.fillStyle = 'black'
					ctx.lineWidth = 1
					ctx.moveTo(0, 0)
					ctx.lineTo(imgWidth, 0)
					ctx.lineTo(imgWidth, imgHeight)
					ctx.lineTo(0, imgHeight)
					ctx.lineTo(0, 0)
					ctx.stroke()

					// Create a close badge in the upper-right corner
					ctx.beginPath()
					ctx.arc(imgWidth - 20, 20, 10, 0, 2 * Math.PI)
					ctx.fillStyle = 'white'
					ctx.fill()
					ctx.lineWidth = 2
					ctx.StrokeStyle = 'darkgray'
					ctx.stroke()
					ctx.beginPath()
					ctx.moveTo(imgWidth - (20 + 5), 20 - 5)
					ctx.lineTo(imgWidth - (20 - 5), 20 + 5)
					ctx.stroke()
					ctx.moveTo(imgWidth - (20 - 5), 20 - 5)
					ctx.lineTo(imgWidth - (20 + 5), 20 + 5)
					ctx.stroke()

					// Add filename to generic icon for non image document
					if (!e.target.result.startsWith('data:image')) {
						ctx.fillStyle = 'black'
						ctx.font = '12px Arial'
						ctx.fillText(file.name, 30, imgHeight - 20)
					}

					// Save miniature
					self.miniatures.push({
						'img': canvas.toDataURL(),
						'coords': String(imgWidth - 20) + ',20,10'
					})

				}

				// Save document
				self.postAttachments.push(e.target.result)

				// Draw a generic icon when document is not an image
				if (e.target.result.startsWith('data:image')) {
					img.src = e.target.result
				} else {
					img.src = OC.generateUrl('svg/core/filetypes/x-office-document?color=d8d8d8')
				}
			}

			// Start reading selected file
			reader.readAsDataURL(file)
		},
		removeAttachment(idx) {
			this.postAttachments.splice(idx, 1)
			this.miniatures.splice(idx, 1)
		},
		getImageMapCoords(idx) {
			return 	this.miniatures[idx].coords
		},
		insert(emoji) {
			if (typeof emoji === 'object') {
				let category = Object.keys(emoji)[0]
				let emojis = emoji[category]
				let firstEmoji = Object.keys(emojis)[0]
				emoji = emojis[firstEmoji]
			}
			this.post += this.$twemoji.parse(emoji) + ' '
			this.$refs.composerInput.innerHTML += this.$twemoji.parse(emoji) + ' '
			this.$refs.emojiPicker.hide()
		},
		togglePopoverMenu() {
			this.menuOpened = !this.menuOpened
		},
		hidePopoverMenu() {
			this.menuOpened = false
		},
		switchType(type) {
			this.type = type
			this.menuOpened = false
			localStorage.setItem('social.lastPostType', type)
		},
		getPostData() {
			let element = this.$refs.composerInput.cloneNode(true)
			Array.from(element.getElementsByClassName('emoji')).forEach((emoji) => {
				var em = document.createTextNode(emoji.getAttribute('alt'))
				emoji.replaceWith(em)
			})

			let contentHtml = element.innerHTML

			// Extract mentions from content and create an array ot of them
			let to = []
			const mentionRegex = /<span class="mention"[^>]+><a[^>]+><img[^>]+>@([\w-_.]+@[\w-.]+)/g
			let match = null
			do {
				match = mentionRegex.exec(contentHtml)
				if (match) {
					to.push(match[1])
				}
			} while (match)

			// Extract hashtags from content and create an array ot of them
			const hashtagRegex = />#([^<]+)</g
			let hashtags = []
			match = null
			do {
				match = hashtagRegex.exec(contentHtml)
				if (match) {
					hashtags.push(match[1])
				}
			} while (match)

			// Remove all html tags but </div> (wich we turn in newlines) and decode the remaining html entities
			let content = contentHtml.replace(/<(?!\/div)[^>]+>/gi, '').replace(/<\/div>/gi, '\n').trim()
			content = he.decode(content)

			let data = {
				content: content,
				to: to,
				hashtags: hashtags,
				type: this.type,
				attachments: this.postAttachments
			}

			if (this.replyTo) {
				data.replyTo = this.replyTo.id
			}

			return data
		},
		keyup(event) {
			if (event.shiftKey || event.ctrlKey) {
				this.createPost(event)
			}
		},
		updatePostFromTribute(event) {
			// Trick to let vue-contenteditable know that tribute replaced a mention or hashtag
			this.$refs.composerInput.oninput(event)
		},
		createPost: async function(event) {

			let postData = this.getPostData()

			// Trick to validate last mention when the user directly clicks on the "post" button without validating it.
			let regex = /@([-\w]+)$/
			let lastMention = postData.content.match(regex)
			if (lastMention) {

				// Ask the server for matching accounts, and wait for the results
				let result = await this.remoteSearchAccounts(lastMention[1])

				// Validate the last mention only when it matches a single account
				if (result.data.result.accounts.length === 1) {
					postData.content = postData.content.replace(regex, '@' + result.data.result.accounts[0].account)
					postData.to.push(result.data.result.accounts[0].account)
				}
			}

			// Abort if the post is a direct message and no valid mentions were found
			if (this.type === 'direct' && postData.to.length === 0) {
				OC.Notification.showTemporary(t('social', 'Error while trying to post your message: Could not find any valid recipients.'), { type: 'error' })
				return
			}

			// Post message
			this.loading = true
			this.$store.dispatch('post', postData).then((response) => {
				this.loading = false
				this.replyTo = null
				this.post = ''
				this.$refs.composerInput.innerText = this.post
				this.postAttachments = []
				this.miniatures = []
				this.$store.dispatch('refreshTimeline')
			})

		},
		remoteSearchAccounts(text) {
			return axios.get(OC.generateUrl('apps/social/api/v1/global/accounts/search?search=' + text))
		},
		remoteSearchHashtags(text) {
			return axios.get(OC.generateUrl('apps/social/api/v1/global/tags/search?search=' + text))
		}
	}
}

</script>
