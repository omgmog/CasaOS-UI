<template>
	<div class="home-section has-text-left">
		<!-- Grouped App List Start -->
		<div v-if="!isLoading">
			<div v-for="group in groupedApps" :key="group.id" class="app-group">
				<!-- Group Header (hidden for default group) -->
				<div v-if="group.id !== 'default'" class="app-group-header is-flex is-align-items-center mb-3 mt-4">
					<span
						v-if="editingGroupId !== group.id"
						class="app-group-name has-text-weight-semibold has-text-grey-100"
						@dblclick="startRenameGroup(group.id)"
					>{{ group.name }}</span>
					<input
						v-else
						ref="groupNameInput"
						v-model="editingGroupName"
						class="app-group-name-input"
						@blur="finishRenameGroup(group.id)"
						@keyup.enter="finishRenameGroup(group.id)"
						@keyup.esc="cancelRenameGroup"
					/>
					<b-dropdown animation="fade1" aria-role="menu" class="ml-2" position="is-bottom-left">
						<template #trigger>
							<b-icon
								class="is-clickable has-text-grey-100"
								icon="dots-vertical-outline"
								pack="casa"
								size="is-16"
							></b-icon>
						</template>
						<b-dropdown-item aria-role="menuitem" @click="startRenameGroup(group.id)">
							{{ $t('Rename') }}
						</b-dropdown-item>
						<b-dropdown-item aria-role="menuitem" class="has-text-red" @click="deleteGroup(group.id)">
							{{ $t('Delete Group') }}
						</b-dropdown-item>
					</b-dropdown>
				</div>

				<!-- Draggable App Grid for this group -->
				<draggable
					v-model="group.apps"
					:draggable="draggable"
					class="app-list contextmenu-canvas"
					tag="div"
					v-bind="groupDragOptions"
					@end="onSortEnd"
					@start="drag = true"
				>
					<div v-for="item in group.apps" :id="'app-' + item.name" :key="'app-' + item.name" class="handle">
						<app-card
							:currentGroupId="group.id"
							:groups="groupsMeta"
							:item="item"
							@configApp="showConfigPanel"
							@createGroupAndMove="onCreateGroupAndMove"
							@importApp="showContainerPanel"
							@moveToGroup="onMoveAppToGroup"
							@updateState="getList"
						></app-card>
					</div>
				</draggable>

				<!-- Empty group placeholder -->
				<div v-if="group.apps.length === 0 && group.id !== 'default'" class="app-group-empty has-text-grey-light has-text-centered py-4">
					{{ $t('Drag apps here') }}
				</div>
			</div>
		</div>

		<!-- Loading skeleton -->
		<div v-else class="app-list contextmenu-canvas">
			<div v-for="index in skCount" :id="'app-' + index" :key="'app-' + index" class="handle">
				<app-card-skeleton :index="index"></app-card-skeleton>
			</div>
		</div>
		<!-- Grouped App List End -->

		<template v-if="oldAppList.length > 0">
			<!-- Title Bar Start -->
			<div class="title-bar is-flex is-align-items-center mt-2rem mb-5">
				<app-section-title-tip
					id="appTitle2"
					class="is-flex-grow-1 has-text-sub-04"
					label="To be rebuilt."
					title="Legacy app (To be rebuilt)."
				>
				</app-section-title-tip>
			</div>
			<!-- Title Bar End -->

			<!-- App List Start -->
			<div class="app-list contextmenu-canvas">
				<!-- Application not imported Start -->
				<div v-for="item in oldAppList" :id="'app-' + item.name" :key="'app-' + item.name" class="handle">
					<app-card
						:isCasa="false"
						:item="item"
						@configApp="showConfigPanel"
						@importApp="showContainerPanel"
						@updateState="getList"
					></app-card>
				</div>
				<!-- Application not imported End -->
			</div>
			<!-- App List End -->
		</template>
	</div>
</template>

<script>
import AppCard from './AppCard.vue'
import AppCardSkeleton from './AppCardSkeleton.vue'
import AppPanel from './AppPanel.vue'
import ExternalLinkPanel from '@/components/Apps/ExternalLinkPanel'
import AppSectionTitleTip from './AppSectionTitleTip.vue'
import draggable from 'vuedraggable'
import concat from 'lodash/concat'
import events from '@/events/events'
import last from 'lodash/last'
import business_ShowNewAppTag from '@/mixins/app/Business_ShowNewAppTag'
import business_LinkApp from '@/mixins/app/Business_LinkApp'
import { ice_i18n } from '@/mixins/base/common-i18n'

const SYNCTHING_STORE_ID = 74

const builtInApplications = [
	{
		id: '1',
		name: 'App Store',
		title: {
			en_us: 'App Store'
		},
		icon: require(`@/assets/img/app/appstore.svg`),
		status: 'running',
		app_type: 'system'
	},
	{
		id: '2',
		name: 'Files',
		title: {
			en_us: 'Files'
		},
		icon: require(`@/assets/img/app/files.svg`),
		status: 'running',
		app_type: 'system'
	}
]

const orderConfig = 'app_order'
const groupsConfig = 'app_groups'

export default {
	mixins: [business_ShowNewAppTag, business_LinkApp],
	data () {
		return {
			groupedApps: [],
			oldAppList: [],
			drag: false,
			isLoading: false,
			draggable: '.handle',
			retryCount: 0,
			appListErrorMessage: '',
			skCount: 0,
			ListRefreshTimer: null,
			editingGroupId: null,
			editingGroupName: ''
		}
	},
	components: {
		AppCard,
		draggable,
		AppSectionTitleTip,
		AppCardSkeleton
	},
	provide () {
		return {
			openAppStore: this.showInstall
		}
	},
	computed: {
		groupDragOptions () {
			return {
				animation: 300,
				group: { name: 'apps' },
				disabled: false,
				ghostClass: 'ghost'
			}
		},
		appList () {
			let all = []
			this.groupedApps.forEach(g => {
				all = all.concat(g.apps)
			})
			return all
		},
		groupsMeta () {
			return this.groupedApps.map(g => ({ id: g.id, name: g.name }))
		}
	},
	created () {
		this.getList()
		this.draggable = this.isMobile() ? '' : '.handle'
		this.$EventBus.$on(events.OPEN_APP_STORE_AND_GOTO_SYNCTHING, () => {
			this.showInstall(SYNCTHING_STORE_ID)
		})

		this.$EventBus.$on(events.RELOAD_APP_LIST, () => {
			this.getList()
		})

		this.ListRefreshTimer = setInterval(() => {
			if (!this.drag) {
				this.getList()
			}
		}, 5000)
	},
	beforeDestroy () {
		this.$EventBus.$off(events.OPEN_APP_STORE_AND_GOTO_SYNCTHING)
		window.removeEventListener('resize', this.getSkCount)

		clearInterval(this.ListRefreshTimer)
	},
	mounted () {
		window.addEventListener('resize', this.getSkCount)
		this.getSkCount()
	},
	methods: {
		isMobile () {
			const userAgent = navigator.userAgent
			const mobileRegex =
				/(phone|pad|pod|iPhone|iPod|ios|iPad|Android|Mobile|BlackBerry|IEMobile|MQQBrowser|JUC|Fennec|wOSBrowser|BrowserNG|WebOS|Symbian|Windows Phone)/i
			const isMobile = mobileRegex.exec(userAgent)
			return isMobile !== null
		},

		getSkCount () {
			const windowWidth = window.innerWidth
			if (windowWidth < 1024) {
				this.skCount = 4
			} else if (windowWidth < 1216) {
				this.skCount = 6
			} else if (windowWidth < 1408) {
				this.skCount = 8
			} else {
				this.skCount = 10
			}
		},

		async getList () {
			if (this.drag) return

			try {
				const orgAppList = await this.$openAPI.appGrid.getAppGrid().then(res => res.data.data || [])
				let orgOldAppList = [],
					orgNewAppList = []
				orgAppList.forEach(item => {
					item.hostname = item.hostname || this.$baseIp
					item.icon = item.icon || require(`@/assets/img/app/default.svg`)
					if (item.app_type === 'v1app' || item.app_type === 'container') {
						orgOldAppList.push(item)
					} else {
						orgNewAppList.push(item)
					}
				})
				this.oldAppList = orgOldAppList

				let listLinkApp = await this.getLinkAppList()
				listLinkApp.forEach(item => {
					item.title = {
						en_us: item.name
					}
				})

				let casaAppList = concat(builtInApplications, orgNewAppList, listLinkApp)

				// Build a map of all apps by name
				const allAppsMap = {}
				casaAppList.forEach(app => {
					allAppsMap[app.name] = app
				})
				const allAppNames = Object.keys(allAppsMap)

				// Try to fetch group data
				let groupData = null
				try {
					const res = await this.$api.users.getCustomStorage(groupsConfig)
					groupData = res.data.data
				} catch (e) {
					// No group data yet
				}

				if (groupData && groupData.version === 1 && Array.isArray(groupData.groups)) {
					// Reconcile: remove uninstalled apps, add new apps to default
					const usedNames = new Set()
					groupData.groups.forEach(group => {
						group.apps = group.apps.filter(name => {
							if (allAppsMap[name]) {
								usedNames.add(name)
								return true
							}
							return false
						})
					})

					// Find new apps not in any group
					const newApps = allAppNames.filter(name => !usedNames.has(name))

					// Ensure default group exists
					let defaultGroup = groupData.groups.find(g => g.id === 'default')
					if (!defaultGroup) {
						defaultGroup = { id: 'default', name: '', apps: [] }
						groupData.groups.unshift(defaultGroup)
					}
					defaultGroup.apps = defaultGroup.apps.concat(newApps)

					// Resolve names to app objects
					this.groupedApps = groupData.groups.map(g => ({
						id: g.id,
						name: g.name,
						apps: g.apps.map(name => allAppsMap[name]).filter(Boolean)
					}))

					// Save if reconciliation changed anything
					if (newApps.length > 0) {
						this.saveGroupData()
					}
				} else {
					// Migration: read app_order and create single default group
					let lateSortList = []
					try {
						const res = await this.$api.users.getCustomStorage(orderConfig)
						lateSortList = res.data.data.data || []
					} catch (e) {
						// No order data
					}

					// Sort apps by existing order
					const propList = casaAppList.map(obj => obj.name)
					const existingList = lateSortList.filter(item => propList.includes(item))
					const futureList = propList.filter(item => !lateSortList.includes(item))
					const sortedNames = existingList.concat(futureList)

					const sortedAppList = casaAppList.sort((obj1, obj2) => {
						return sortedNames.indexOf(obj1.name) - sortedNames.indexOf(obj2.name)
					})

					this.groupedApps = [{
						id: 'default',
						name: '',
						apps: sortedAppList
					}]

					// Save as new group format
					this.saveGroupData()
				}

				this.isLoading = false
				this.retryCount = 0
				this.appListErrorMessage = ''

				// Also update app_order for backward compat
				this.saveSortData()
			} catch (error) {
				console.error(error)
				this.isLoading = true
				if (this.retryCount < 5) {
					setTimeout(() => {
						this.retryCount++
						this.getList()
					}, 2000)
				} else {
					this.appListErrorMessage = 'Failed to get app list.'
					this.$buefy.toast.open({
						message: this.$t(`Failed to load apps, please refresh later.`),
						type: 'is-danger'
					})
				}
			}
		},

		saveGroupData () {
			const data = {
				version: 1,
				groups: this.groupedApps.map(g => ({
					id: g.id,
					name: g.name,
					apps: g.apps.map(app => app.name)
				}))
			}
			this.$api.users.setCustomStorage(groupsConfig, data)
		},

		saveSortData () {
			let newList = this.appList.map(item => item.name)
			let data = {
				data: newList
			}
			this.$api.users.setCustomStorage(orderConfig, data)
		},

		onSortEnd () {
			this.drag = false
			this.saveGroupData()
			this.saveSortData()
		},

		createGroup () {
			this.$buefy.dialog.prompt({
				title: this.$t('Create Group'),
				message: this.$t('Group name'),
				inputAttrs: {
					placeholder: this.$t('Group name'),
					maxlength: 50
				},
				confirmText: this.$t('Create'),
				cancelText: this.$t('Cancel'),
				trapFocus: true,
				onConfirm: (value) => {
					if (!value || !value.trim()) return
					const newGroup = {
						id: 'g_' + Date.now(),
						name: value.trim(),
						apps: []
					}
					this.groupedApps.push(newGroup)
					this.saveGroupData()
				}
			})
		},

		deleteGroup (groupId) {
			if (groupId === 'default') return

			const group = this.groupedApps.find(g => g.id === groupId)
			if (!group) return

			this.$buefy.dialog.confirm({
				title: this.$t('Delete Group'),
				message: this.$t('Apps in this group will be moved to ungrouped.'),
				confirmText: this.$t('Delete Group'),
				cancelText: this.$t('Cancel'),
				type: 'is-danger',
				onConfirm: () => {
					const defaultGroup = this.groupedApps.find(g => g.id === 'default')
					if (defaultGroup) {
						defaultGroup.apps = defaultGroup.apps.concat(group.apps)
					}
					this.groupedApps = this.groupedApps.filter(g => g.id !== groupId)
					this.saveGroupData()
					this.saveSortData()
				}
			})
		},

		startRenameGroup (groupId) {
			const group = this.groupedApps.find(g => g.id === groupId)
			if (!group || groupId === 'default') return
			this.editingGroupId = groupId
			this.editingGroupName = group.name
			this.$nextTick(() => {
				if (this.$refs.groupNameInput) {
					const input = Array.isArray(this.$refs.groupNameInput)
						? this.$refs.groupNameInput[0]
						: this.$refs.groupNameInput
					if (input) input.focus()
				}
			})
		},

		finishRenameGroup (groupId) {
			const group = this.groupedApps.find(g => g.id === groupId)
			if (group && this.editingGroupName.trim()) {
				group.name = this.editingGroupName.trim()
				this.saveGroupData()
			}
			this.editingGroupId = null
			this.editingGroupName = ''
		},

		cancelRenameGroup () {
			this.editingGroupId = null
			this.editingGroupName = ''
		},

		onMoveAppToGroup (appName, targetGroupId) {
			// Find source group
			let sourceGroup = null
			let appObj = null
			for (const group of this.groupedApps) {
				const idx = group.apps.findIndex(a => a.name === appName)
				if (idx !== -1) {
					sourceGroup = group
					appObj = group.apps[idx]
					group.apps.splice(idx, 1)
					break
				}
			}
			if (!appObj) return

			const targetGroup = this.groupedApps.find(g => g.id === targetGroupId)
			if (targetGroup) {
				targetGroup.apps.push(appObj)
			}
			this.saveGroupData()
			this.saveSortData()
		},

		onCreateGroupAndMove (appName) {
			this.$buefy.dialog.prompt({
				title: this.$t('Create Group'),
				message: this.$t('Group name'),
				inputAttrs: {
					placeholder: this.$t('Group name'),
					maxlength: 50
				},
				confirmText: this.$t('Create'),
				cancelText: this.$t('Cancel'),
				trapFocus: true,
				onConfirm: (value) => {
					if (!value || !value.trim()) return
					const newGroup = {
						id: 'g_' + Date.now(),
						name: value.trim(),
						apps: []
					}
					this.groupedApps.push(newGroup)
					this.onMoveAppToGroup(appName, newGroup.id)
				}
			})
		},

		async showInstall (storeId = 0, mode = '') {
			if (mode === 'custom') {
				this.$messageBus('apps_custominstall')
			}

			const networks = await this.$api.container.getNetworks()
			const memory = this.$store.state.hardwareInfo.mem
			const configData = {
				networks: networks.data.data,
				memory: memory
			}
			this.$buefy.modal.open({
				parent: this,
				component: AppPanel,
				hasModalCard: true,
				customClass: 'app-panel',
				trapFocus: true,
				canCancel: ['escape'],
				scroll: 'keep',
				animation: 'zoom-in',
				events: {
					updateState: () => {
						this.getList()
					}
				},
				props: {
					id: '0',
					state: 'install',
					configData: configData,
					storeId: storeId,
					settingData: mode !== 'custom' ? undefined : {}
				}
			})
		},

		async showConfigPanel (item, isCasa) {
			let name = item.name
			this.$messageBus('appsexsiting_open', name)
			try {
				if (item?.app_type === 'LinkApp') {
					await this.showExternalLinkPanel(item)
					return
				}
				const networks = await this.$api.container.getNetworks()
				const memory = this.$store.state.hardwareInfo.mem
				const configData = {
					networks: networks.data.data,
					memory: memory
				}
				const ret = await this.$openAPI.appManagement.compose.myComposeApp(name, {
					headers: {
						'content-type': 'application/yaml',
						accept: 'application/yaml'
					}
				})
				this.$buefy.modal.open({
					parent: this,
					component: AppPanel,
					hasModalCard: true,
					customClass: '',
					trapFocus: true,
					canCancel: [''],
					scroll: 'keep',
					animation: 'zoom-in',
					events: {
						updateState: () => {
							this.getList()
						}
					},
					props: {
						id: name,
						state: 'update',
						isCasa: isCasa,
						runningStatus: item.status,
						configData: configData,
						settingComposeData: ret.data
					}
				})
			} catch (e) {
				console.error(e)
			}
		},

		async showContainerPanel (item) {
			this.$messageBus('appsexsiting_open', item.name)
			let id = item.name
			const networks = await this.$api.container.getNetworks()
			const memory = this.$store.state.hardwareInfo.mem
			const configData = {
				networks: networks.data.data,
				memory: memory
			}
			const ret = await this.$api.container.getInfo(id)
			this.$buefy.modal.open({
				parent: this,
				component: AppPanel,
				hasModalCard: true,
				customClass: '',
				trapFocus: true,
				canCancel: [''],
				scroll: 'keep',
				animation: 'zoom-in',
				events: {
					updateState: () => {
						this.getList()
					}
				},
				props: {
					id: id,
					state: 'update',
					isCasa: false,
					runningStatus: item.status,
					configData: configData,
					settingData: ret.data.data
				}
			})
		},

		async showExternalLinkPanel (item = {}) {
			this.$buefy.modal.open({
				parent: this,
				component: ExternalLinkPanel,
				hasModalCard: true,
				customClass: '',
				trapFocus: true,
				canCancel: [''],
				scroll: 'keep',
				animation: 'zoom-in',
				events: {
					updateState: () => {
						this.$messageBus('apps_external')
						this.getList().then(() => {
							this.scrollToNewApp()
						})
					}
				},
				props: {
					linkName: item.name,
					linkHost: item.hostname,
					linkIcon: item.icon
				}
			})
		},

		scrollToNewApp () {
			let name = last(this.newAppIds)
			let showEl = document.getElementById('app-' + name)
			showEl?.scrollIntoView({ behavior: 'smooth', block: 'end' })
		},

		messageBusToast (message, type) {
			let duration = 5000
			this.$buefy.toast.open({
				message: message,
				duration,
				type
			})
		}
	},
	sockets: {
		'app:install-end' () {
			this.getList().then(() => {
				this.scrollToNewApp()
			})
		},
		'app:install-error' () {
			this.getList().then(() => {
				this.scrollToNewApp()
			})
		},
		'app:uninstall-end' () {
			this.getList()
		},
		'app:apply-changes-error' (res) {
			this.messageBusToast(res.Properties.message, 'is-danger')
		},
		'app:apply-changes-end' (res) {
			let languages = JSON.parse(res.Properties['app:title'])
			const title = ice_i18n(languages)
			this.messageBusToast(title + ' is OK', 'is-success')

			this.addIdToSessionStorage(res.Properties['app:name'])

			this.getList().then(() => {
				this.scrollToNewApp()
			})
		},
		'app:update-end' (data) {
			if (data.Properties['docker:image:updated'] === 'true') {
				this.addIdToSessionStorage(data.Properties['app:name'])

				this.$buefy.toast.open({
					message: this.$t(`{name} has been updated to the latest version!`, {
						name: data.Properties.name
					}),
					type: 'is-success'
				})
				this.getList().then(() => {
					this.scrollToNewApp()
				})
			}
		},
		'app:update-error' (data) {
			if (data.Properties.cid === this.item.id) {
				this.isUpdating = false
				this.$buefy.toast.open({
					message: this.$t(data.Properties['error']),
					type: 'is-danger'
				})
			}
		}
	}
}
</script>

<style lang="scss" scoped>
.app-list {
	position: relative;
	display: grid;
	gap: 1rem;
	min-height: 2rem;

	grid-template-columns: repeat(auto-fill, minmax(0, 160px));
}

.app-group {
	margin-bottom: 0.5rem;
}

.app-group-header {
	padding: 0.25rem 0;
}

.app-group-name {
	font-size: 0.9rem;
	cursor: default;
	user-select: none;
}

.app-group-name-input {
	font-size: 0.9rem;
	font-weight: 600;
	border: 1px solid hsla(208, 16%, 84%, 1);
	border-radius: 4px;
	padding: 0.1rem 0.4rem;
	outline: none;
	background: transparent;

	&:focus {
		border-color: hsla(208, 100%, 45%, 1);
	}
}

.app-group-empty {
	border: 2px dashed hsla(208, 16%, 88%, 1);
	border-radius: 8px;
	font-size: 0.85rem;
}
</style>
