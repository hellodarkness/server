<!--
  - @copyright Copyright (c) 2023 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license AGPL-3.0-or-later
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
	<tr :class="{'files-list__row--dragover': dragover, 'files-list__row--loading': isLoading}"
		data-cy-files-list-row
		:data-cy-files-list-row-fileid="fileid"
		:data-cy-files-list-row-name="source.basename"
		:draggable="canDrag"
		class="files-list__row"
		v-on="rowListeners">
		<!-- Failed indicator -->
		<span v-if="source.attributes.failed" class="files-list__row--failed" />

		<!-- Checkbox -->
		<FileEntryCheckbox :display-name="displayName"
			:fileid="fileid"
			:is-loading="isLoading"
			:nodes="nodes" />

		<!-- Link to file -->
		<td class="files-list__row-name" data-cy-files-list-row-name>
			<!-- Icon or preview -->
			<FileEntryPreview ref="preview"
				:source="source"
				:dragover="dragover"
				@click.native="execDefaultAction" />

			<FileEntryName ref="name"
				:display-name="displayName"
				:extension="extension"
				:files-list-width="filesListWidth"
				:nodes="nodes"
				:source="source"
				@click="execDefaultAction" />
		</td>

		<!-- Actions -->
		<FileEntryActions v-show="!isRenamingSmallScreen"
			ref="actions"
			:class="`files-list__row-actions-${uniqueId}`"
			:files-list-width="filesListWidth"
			:loading.sync="loading"
			:opened.sync="openedMenu"
			:source="source" />

		<!-- Size -->
		<td v-if="!compact && isSizeAvailable"
			:style="sizeOpacity"
			class="files-list__row-size"
			data-cy-files-list-row-size
			@click="openDetailsIfAvailable">
			<span>{{ size }}</span>
		</td>

		<!-- Mtime -->
		<td v-if="!compact && isMtimeAvailable"
			:style="mtimeOpacity"
			class="files-list__row-mtime"
			data-cy-files-list-row-mtime
			@click="openDetailsIfAvailable">
			<NcDateTime :timestamp="source.mtime" :ignore-seconds="true" />
		</td>

		<!-- View columns -->
		<td v-for="column in columns"
			:key="column.id"
			:class="`files-list__row-${currentView?.id}-${column.id}`"
			class="files-list__row-column-custom"
			:data-cy-files-list-row-column-custom="column.id"
			@click="openDetailsIfAvailable">
			<CustomElementRender :current-view="currentView"
				:render="column.render"
				:source="source" />
		</td>
	</tr>
</template>

<script lang="ts">
import type { PropType } from 'vue'

import { extname, join } from 'path'
import { FileType, formatFileSize, Permission, Folder, File as NcFile, NodeStatus, Node, View } from '@nextcloud/files'
import { getUploader } from '@nextcloud/upload'
import { showError } from '@nextcloud/dialogs'
import { translate as t } from '@nextcloud/l10n'
import { vOnClickOutside } from '@vueuse/components'
import moment from '@nextcloud/moment'
import { generateUrl } from '@nextcloud/router'
import Vue, { defineComponent } from 'vue'

import { action as sidebarAction } from '../actions/sidebarAction.ts'
import { getDragAndDropPreview } from '../utils/dragUtils.ts'
import { handleCopyMoveNodeTo } from '../actions/moveOrCopyAction.ts'
import { hashCode } from '../utils/hashUtils.ts'
import { MoveCopyAction } from '../actions/moveOrCopyActionUtils.ts'
import { useActionsMenuStore } from '../store/actionsmenu.ts'
import { useDragAndDropStore } from '../store/dragging.ts'
import { useFilesStore } from '../store/files.ts'
import { useRenamingStore } from '../store/renaming.ts'
import { useSelectionStore } from '../store/selection.ts'
import NcDateTime from '@nextcloud/vue/dist/Components/NcDateTime.js'
import CustomElementRender from './CustomElementRender.vue'
import FileEntryActions from './FileEntry/FileEntryActions.vue'
import FileEntryCheckbox from './FileEntry/FileEntryCheckbox.vue'
import FileEntryName from './FileEntry/FileEntryName.vue'
import FileEntryPreview from './FileEntry/FileEntryPreview.vue'
import logger from '../logger.js'

Vue.directive('onClickOutside', vOnClickOutside)

export default defineComponent({
	name: 'FileEntry',

	components: {
		CustomElementRender,
		FileEntryActions,
		FileEntryCheckbox,
		FileEntryName,
		FileEntryPreview,
		NcDateTime,
	},

	props: {
		isMtimeAvailable: {
			type: Boolean,
			default: false,
		},
		isSizeAvailable: {
			type: Boolean,
			default: false,
		},
		source: {
			type: [Folder, NcFile, Node] as PropType<Node>,
			required: true,
		},
		nodes: {
			type: Array as PropType<Node[]>,
			required: true,
		},
		filesListWidth: {
			type: Number,
			default: 0,
		},
		compact: {
			type: Boolean,
			default: false,
		},
	},

	setup() {
		const actionsMenuStore = useActionsMenuStore()
		const draggingStore = useDragAndDropStore()
		const filesStore = useFilesStore()
		const renamingStore = useRenamingStore()
		const selectionStore = useSelectionStore()
		return {
			actionsMenuStore,
			draggingStore,
			filesStore,
			renamingStore,
			selectionStore,
		}
	},

	data() {
		return {
			loading: '',
			dragover: false,
		}
	},

	computed: {
		/**
		 * Conditionally add drag and drop listeners
		 * Do not add drag start and over listeners on renaming to allow to drag and drop text
		 */
		rowListeners() {
			const conditionals = this.isRenaming
				? {}
				: {
					dragstart: this.onDragStart,
					dragover: this.onDragOver,
				}

			return {
				...conditionals,
				contextmenu: this.onRightClick,
				dragleave: this.onDragLeave,
				dragend: this.onDragEnd,
				drop: this.onDrop,
			}
		},
		currentView(): View {
			return this.$navigation.active as View
		},
		columns() {
			// Hide columns if the list is too small
			if (this.filesListWidth < 512 || this.compact) {
				return []
			}
			return this.currentView?.columns || []
		},

		currentDir() {
			// Remove any trailing slash but leave root slash
			return (this.$route?.query?.dir?.toString() || '/').replace(/^(.+)\/$/, '$1')
		},
		currentFileId() {
			return this.$route.params?.fileid || this.$route.query?.fileid || null
		},
		fileid() {
			return this.source?.fileid?.toString?.()
		},
		uniqueId() {
			return hashCode(this.source.source)
		},
		isLoading() {
			return this.source.status === NodeStatus.LOADING
		},

		extension() {
			if (this.source.attributes?.displayName) {
				return extname(this.source.attributes.displayName)
			}
			return this.source.extension || ''
		},
		displayName() {
			const ext = this.extension
			const name = (this.source.attributes.displayName
				|| this.source.basename)

			// Strip extension from name if defined
			return !ext ? name : name.slice(0, 0 - ext.length)
		},

		size() {
			const size = parseInt(this.source.size, 10) || 0
			if (typeof size !== 'number' || size < 0) {
				return t('files', 'Pending')
			}
			return formatFileSize(size, true)
		},
		sizeOpacity() {
			const maxOpacitySize = 10 * 1024 * 1024

			const size = parseInt(this.source.size, 10) || 0
			if (!size || size < 0) {
				return {}
			}

			const ratio = Math.round(Math.min(100, 100 * Math.pow((this.source.size / maxOpacitySize), 2)))
			return {
				color: `color-mix(in srgb, var(--color-main-text) ${ratio}%, var(--color-text-maxcontrast))`,
			}
		},
		mtimeOpacity() {
			const maxOpacityTime = 31 * 24 * 60 * 60 * 1000 // 31 days

			const mtime = this.source.mtime?.getTime?.()
			if (!mtime) {
				return {}
			}

			// 1 = today, 0 = 31 days ago
			const ratio = Math.round(Math.min(100, 100 * (maxOpacityTime - (Date.now() - mtime)) / maxOpacityTime))
			if (ratio < 0) {
				return {}
			}
			return {
				color: `color-mix(in srgb, var(--color-main-text) ${ratio}%, var(--color-text-maxcontrast))`,
			}
		},
		mtimeTitle() {
			if (this.source.mtime) {
				return moment(this.source.mtime).format('LLL')
			}
			return ''
		},

		draggingFiles() {
			return this.draggingStore.dragging
		},
		selectedFiles() {
			return this.selectionStore.selected
		},
		isSelected() {
			return this.selectedFiles.includes(this.fileid)
		},

		isRenaming() {
			return this.renamingStore.renamingNode === this.source
		},
		isRenamingSmallScreen() {
			return this.isRenaming && this.filesListWidth < 512
		},

		isActive() {
			return this.fileid === this.currentFileId?.toString?.()
		},

		canDrag() {
			if (this.isRenaming) {
				return false
			}

			const canDrag = (node: Node): boolean => {
				return (node?.permissions & Permission.UPDATE) !== 0
			}

			// If we're dragging a selection, we need to check all files
			if (this.selectedFiles.length > 0) {
				const nodes = this.selectedFiles.map(fileid => this.filesStore.getNode(fileid)) as Node[]
				return nodes.every(canDrag)
			}
			return canDrag(this.source)
		},

		canDrop() {
			if (this.source.type !== FileType.Folder) {
				return false
			}

			// If the current folder is also being dragged, we can't drop it on itself
			if (this.draggingFiles.includes(this.fileid)) {
				return false
			}

			return (this.source.permissions & Permission.CREATE) !== 0
		},

		openedMenu: {
			get() {
				return this.actionsMenuStore.opened === this.uniqueId
			},
			set(opened) {
				// Only reset when opening a new menu
				if (opened) {
					// Reset any right click position override on close
					// Wait for css animation to be done
					const root = this.$root.$el as HTMLElement
					root.style.removeProperty('--mouse-pos-x')
					root.style.removeProperty('--mouse-pos-y')
				}

				this.actionsMenuStore.opened = opened ? this.uniqueId : null
			},
		},
	},

	watch: {
		/**
		 * When the source changes, reset the preview
		 * and fetch the new one.
		 */
		source() {
			this.resetState()
		},
	},

	beforeDestroy() {
		this.resetState()
	},

	methods: {
		resetState() {
			// Reset loading state
			this.loading = ''

			this.$refs.preview.reset()

			// Close menu
			this.openedMenu = false
		},

		// Open the actions menu on right click
		onRightClick(event) {
			// If already opened, fallback to default browser
			if (this.openedMenu) {
				return
			}

			const root = this.$root.$el as HTMLElement
			const contentRect = root.getBoundingClientRect()
			// Using Math.min/max to prevent the menu from going out of the AppContent
			// 200 = max width of the menu
			root.style.setProperty('--mouse-pos-x', Math.max(contentRect.left, Math.min(event.clientX, event.clientX - 200)) + 'px')
			root.style.setProperty('--mouse-pos-y', Math.max(contentRect.top, event.clientY - contentRect.top) + 'px')

			// If the clicked row is in the selection, open global menu
			const isMoreThanOneSelected = this.selectedFiles.length > 1
			this.actionsMenuStore.opened = this.isSelected && isMoreThanOneSelected ? 'global' : this.uniqueId

			// Prevent any browser defaults
			event.preventDefault()
			event.stopPropagation()
		},

		execDefaultAction(event) {
			if (event.ctrlKey || event.metaKey) {
				event.preventDefault()
				window.open(generateUrl('/f/{fileId}', { fileId: this.fileid }))
				return false
			}

			this.$refs.actions.execDefaultAction(event)
		},

		openDetailsIfAvailable(event) {
			event.preventDefault()
			event.stopPropagation()
			if (sidebarAction?.enabled?.([this.source], this.currentView)) {
				sidebarAction.exec(this.source, this.currentView, this.currentDir)
			}
		},

		onDragOver(event: DragEvent) {
			this.dragover = this.canDrop
			if (!this.canDrop) {
				event.dataTransfer.dropEffect = 'none'
				return
			}

			// Handle copy/move drag and drop
			if (event.ctrlKey) {
				event.dataTransfer.dropEffect = 'copy'
			} else {
				event.dataTransfer.dropEffect = 'move'
			}
		},
		onDragLeave(event: DragEvent) {
			// Counter bubbling, make sure we're ending the drag
			// only when we're leaving the current element
			const currentTarget = event.currentTarget as HTMLElement
			if (currentTarget?.contains(event.relatedTarget as HTMLElement)) {
				return
			}

			this.dragover = false
		},

		async onDragStart(event: DragEvent) {
			event.stopPropagation()
			if (!this.canDrag) {
				event.preventDefault()
				event.stopPropagation()
				return
			}

			logger.debug('Drag started')

			// Reset any renaming
			this.renamingStore.$reset()

			// Dragging set of files, if we're dragging a file
			// that is already selected, we use the entire selection
			if (this.selectedFiles.includes(this.fileid)) {
				this.draggingStore.set(this.selectedFiles)
			} else {
				this.draggingStore.set([this.fileid])
			}

			const nodes = this.draggingStore.dragging
				.map(fileid => this.filesStore.getNode(fileid)) as Node[]

			const image = await getDragAndDropPreview(nodes)
			event.dataTransfer?.setDragImage(image, -10, -10)
		},
		onDragEnd() {
			this.draggingStore.reset()
			this.dragover = false
			logger.debug('Drag ended')
		},

		async onDrop(event: DragEvent) {
			// skip if native drop like text drag and drop from files names
			if (!this.draggingFiles && !event.dataTransfer?.files?.length) {
				return
			}

			event.preventDefault()
			event.stopPropagation()

			// If another button is pressed, cancel it
			// This allows cancelling the drag with the right click
			if (!this.canDrop || event.button !== 0) {
				return
			}

			const isCopy = event.ctrlKey
			this.dragover = false

			logger.debug('Dropped', { event, selection: this.draggingFiles })

			// Check whether we're uploading files
			if (event.dataTransfer?.files?.length > 0) {
				const uploader = getUploader()
				event.dataTransfer.files.forEach((file: File) => {
					uploader.upload(join(this.source.path, file.name), file)
				})
				logger.debug(`Uploading files to ${this.source.path}`)
				return
			}

			const nodes = this.draggingFiles.map(fileid => this.filesStore.getNode(fileid)) as Node[]
			nodes.forEach(async (node: Node) => {
				Vue.set(node, 'status', NodeStatus.LOADING)
				try {
					// TODO: resolve potential conflicts prior and force overwrite
					await handleCopyMoveNodeTo(node, this.source, isCopy ? MoveCopyAction.COPY : MoveCopyAction.MOVE)
				} catch (error) {
					logger.error('Error while moving file', { error })
					if (isCopy) {
						showError(t('files', 'Could not copy {file}. {message}', { file: node.basename, message: error.message || '' }))
					} else {
						showError(t('files', 'Could not move {file}. {message}', { file: node.basename, message: error.message || '' }))
					}
				} finally {
					Vue.set(node, 'status', undefined)
				}
			})

			// Reset selection after we dropped the files
			// if the dropped files are within the selection
			if (this.draggingFiles.some(fileid => this.selectedFiles.includes(fileid))) {
				logger.debug('Dropped selection, resetting select store...')
				this.selectionStore.reset()
			}
		},

		t,
		formatFileSize,
	},
})
</script>
