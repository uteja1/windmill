<script lang="ts">
	import MenuLink from './MenuLink.svelte'
	import {
		superadmin,
		usedTriggerKinds,
		userStore,
		workspaceStore,
		isCriticalAlertsUIOpen,
		enterpriseLicense,
		devopsRole
	} from '$lib/stores'
	import { SIDEBAR_SHOW_SCHEDULES } from '$lib/consts'
	import {
		BookOpen,
		Bot,
		Boxes,
		Calendar,
		DollarSign,
		Eye,
		Logs,
		FolderCog,
		FolderOpen,
		Github,
		HelpCircle,
		Home,
		LogOut,
		Newspaper,
		Play,
		Route,
		ServerCog,
		Settings,
		UserCog,
		Plus,
		Unplug,
		AlertCircle,
		Database
	} from 'lucide-svelte'
	import Menu from '../common/menu/MenuV2.svelte'
	import MenuButton from './MenuButton.svelte'
	import { MenuItem } from '@rgossiaux/svelte-headlessui'
	import UserMenu from './UserMenu.svelte'
	import DiscordIcon from '../icons/brands/Discord.svelte'
	import { WorkspaceService } from '$lib/gen'
	import { sendUserToast } from '$lib/toast'
	import { clearStores } from '$lib/storeUtils'
	import { goto } from '$lib/navigation'
	import ConfirmationModal from '../common/confirmationModal/ConfirmationModal.svelte'
	import { twMerge } from 'tailwind-merge'
	import { onMount } from 'svelte'
	import { base } from '$lib/base'
	import { type Changelog, changelogs } from './changelogs'
	import { page } from '$app/stores'
	import SideBarNotification from './SideBarNotification.svelte'
	import KafkaIcon from '../icons/KafkaIcon.svelte'
	import NatsIcon from '../icons/NatsIcon.svelte'

	export let numUnacknowledgedCriticalAlerts = 0

	$: mainMenuLinks = [
		{ label: 'Home', href: `${base}/`, icon: Home },
		{ label: 'Runs', href: `${base}/runs`, icon: Play },
		{
			label: 'Variables',
			href: `${base}/variables`,
			icon: DollarSign,
			disabled: $userStore?.operator
		},
		{ label: 'Resources', href: `${base}/resources`, icon: Boxes, disabled: $userStore?.operator }
	]

	$: triggerMenuLinks = [
		{
			label: 'Schedules',
			href: `${base}/schedules`,
			icon: Calendar,
			disabled: !SIDEBAR_SHOW_SCHEDULES || $userStore?.operator
		},
		...defaultExtraTriggerLinks.filter(
			(link) => $usedTriggerKinds.includes(link.kind) || $page.url.pathname.includes(link.href)
		)
	]
	async function leaveWorkspace() {
		await WorkspaceService.leaveWorkspace({ workspace: $workspaceStore ?? '' })
		sendUserToast('You left the workspace')
		clearStores()
		goto('/user/workspaces')
	}

	$: defaultExtraTriggerLinks = [
		{
			label: 'HTTP',
			href: '/routes',
			icon: Route,
			disabled: $userStore?.operator,
			kind: 'http'
		},
		{
			label: 'WebSockets',
			href: '/websocket_triggers',
			icon: Unplug,
			disabled: $userStore?.operator,
			kind: 'ws'
		},
		{
			label: 'Postgres',
			href: '/postgres_triggers',
			icon: Database,
			disabled: $userStore?.operator,
			kind: 'postgres'
		},
		{
			label: 'Kafka' + ($enterpriseLicense ? '' : ' (EE)'),
			href: '/kafka_triggers',
			icon: KafkaIcon,
			disabled: $userStore?.operator || !$enterpriseLicense,
			kind: 'kafka'
		},
		{
			label: 'NATS' + ($enterpriseLicense ? '' : ' (EE)'),
			href: '/nats_triggers',
			icon: NatsIcon,
			disabled: $userStore?.operator || !$enterpriseLicense,
			kind: 'nats'
		}
	]

	$: extraTriggerLinks = defaultExtraTriggerLinks.filter((link) => {
		return !$page.url.pathname.includes(link.href) && !$usedTriggerKinds.includes(link.kind)
	})
	$: secondaryMenuLinks = [
		// {
		// 	label: 'Workspace',
		// 	href: '/workspace_settings',
		// 	icon: Cog,
		// 	disabled: !$userStore?.is_admin && !$userStore?.is_super_admin
		// },
		{
			label: 'Settings',
			icon: Settings,
			subItems: [
				{
					label: 'Account',
					href: '#user-settings',
					icon: UserCog,
					faIcon: undefined
				},
				...($userStore?.is_admin || $superadmin
					? [
							{
								label: 'Workspace',
								href: `${base}/workspace_settings`,
								icon: FolderCog,
								faIcon: undefined
							}
					  ]
					: []),
				...($superadmin
					? [
							{
								label: 'Instance',
								href: '#superadmin-settings',
								icon: ServerCog,
								faIcon: undefined
							}
					  ]
					: []),
				...(!$superadmin && !$userStore?.is_admin
					? [
							{
								label: 'Leave workspace',
								action: () => {
									leaveWorkspaceModal = true
								},
								class: 'text-red-400',
								icon: LogOut,
								faIcon: undefined
							}
					  ]
					: [])
			],
			disabled: $userStore?.operator
		},
		{ label: 'Workers', href: `${base}/workers`, icon: Bot, disabled: $userStore?.operator },
		{
			label: 'Folders & Groups',
			icon: FolderOpen,
			subItems: [
				{
					label: 'Folders',
					href: `${base}/folders`,
					icon: FolderOpen,
					disabled: $userStore?.operator,
					faIcon: undefined
				},
				{
					label: 'Groups',
					href: `${base}/groups`,
					icon: UserCog,
					disabled: $userStore?.operator,
					faIcon: undefined
				}
			],
			disabled: $userStore?.operator
		},
		$devopsRole || $userStore?.is_admin
			? {
					label: 'Logs',
					icon: Logs,
					subItems: [
						{
							label: 'Audit logs',
							href: `${base}/audit_logs`,
							icon: Eye
						},
						...($devopsRole
							? [
									{
										label: 'Service logs',
										href: `${base}/service_logs`,
										icon: Logs
									}
							  ]
							: []),
						...($enterpriseLicense
							? [
									{
										label: 'Critical alerts',
										action: () => {
											isCriticalAlertsUIOpen.set(true)
										},
										icon: AlertCircle,
										notificationCount: numUnacknowledgedCriticalAlerts
									}
							  ]
							: [])
					]
			  }
			: {
					label: 'Audit logs',
					href: `${base}/audit_logs`,
					icon: Eye,
					disabled: $userStore?.operator
			  }
	]

	let hasNewChangelogs = false
	let recentChangelogs: Changelog[] = []
	let lastOpened = localStorage.getItem('changelogsLastOpened')

	onMount(() => {
		if (lastOpened) {
			// @ts-ignore
			recentChangelogs = changelogs.filter((changelog) => changelog.date > lastOpened)
			hasNewChangelogs =
				recentChangelogs.length > 0 && lastOpened !== new Date().toISOString().split('T')[0]
		} else {
			recentChangelogs = changelogs.slice(-3)
		}
	})

	function openChangelogs() {
		const today = new Date().toISOString().split('T')[0]
		localStorage.setItem('changelogsLastOpened', today)
		hasNewChangelogs = false
	}

	const thirdMenuLinks = [
		{
			label: 'Help',
			icon: HelpCircle,
			subItems: [
				{ label: 'Docs', href: 'https://www.windmill.dev/docs/intro/', icon: BookOpen },
				{
					label: 'Feedbacks',
					href: 'https://discord.gg/V7PM2YHsPB',
					icon: DiscordIcon
				},
				{
					label: 'Issues',
					href: 'https://github.com/windmill-labs/windmill/issues/new',
					icon: Github
				},
				{
					label: 'Changelog',
					href: 'https://www.windmill.dev/changelog/',
					icon: Newspaper
				}
			]
		}
	]

	export let isCollapsed: boolean = false
	export let noGap: boolean = false

	let leaveWorkspaceModal = false

	function computeAllNotificationsCount(menuItems: any[]) {
		let count = 0
		for (const menuItem of menuItems) {
			count += menuItem?.['notificationCount'] ?? 0
		}
		return count
	}
</script>

<nav
	class={twMerge(
		'grow flex flex-col overflow-x-hidden scrollbar-hidden px-2 md:pb-2 justify-between',
		noGap ? 'gap-0' : 'gap-16'
	)}
>
	<div class={twMerge('pt-4 ', noGap ? 'md:mb-0 mb-0' : 'mb-6 md:mb-10')}>
		<div class="space-y-1">
			{#each mainMenuLinks as menuLink (menuLink.href ?? menuLink.label)}
				<MenuLink class="!text-xs" {...menuLink} {isCollapsed} />
			{/each}
		</div>
		<div class="pt-4">
			<div
				class="text-gray-400 text-[0.5rem] uppercase transition-opacity"
				class:opacity-0={isCollapsed}>Triggers</div
			>
			<div class="space-y-1">
				{#each triggerMenuLinks as menuLink (menuLink.href ?? menuLink.label)}
					<MenuLink class="!text-xs" {...menuLink} {isCollapsed} />
				{/each}
				{#if extraTriggerLinks.length > 0 && !$userStore?.operator}
					<Menu>
						<div
							slot="trigger"
							class="w-full text-gray-400 text-2xs flex flex-row gap-1 py-1 items-center px-2 hover:bg-[#2A3648] dark:hover:bg-[#30404e] rounded"
						>
							<Plus size={14} />
						</div>
						{#each extraTriggerLinks as subItem (subItem.href ?? subItem.label)}
							<MenuItem>
								<div class="py-1" role="none">
									<a
										href={subItem.disabled ? '' : subItem.href}
										class={twMerge(
											'text-secondary block px-4 py-2 text-2xs hover:bg-surface-hover hover:text-primary',
											subItem.disabled ? 'pointer-events-none opacity-50' : ''
										)}
										role="menuitem"
										tabindex="-1"
									>
										<div class="flex flex-row items-center gap-2">
											{#if subItem.icon}
												<svelte:component this={subItem.icon} size={16} />
											{/if}
											{subItem.label}
										</div>
									</a>
								</div>
							</MenuItem>
						{/each}
					</Menu>
				{/if}
			</div>
		</div>
	</div>
	<div class="flex flex-col h-full justify-end">
		<div class={twMerge('space-y-0.5 mb-6 md:mb-10', noGap ? 'md:mb-0 mb-0' : 'mb-6 md:mb-10')}>
			<UserMenu {isCollapsed} />
			{#each secondaryMenuLinks as menuLink (menuLink.href ?? menuLink.label)}
				{#if menuLink.subItems}
					{@const notificationsCount = computeAllNotificationsCount(menuLink.subItems)}
					<Menu>
						<div slot="trigger">
							<MenuButton class="!text-2xs" {...menuLink} {isCollapsed} {notificationsCount} />
						</div>
						{#each menuLink.subItems as subItem (subItem.href ?? subItem.label)}
							<MenuItem>
								<div class="py-1" role="none">
									{#if subItem?.['action']}
										<button
											class="text-secondary font-normal w-full block px-4 py-2 text-2xs hover:bg-surface-hover hover:text-primary"
											on:click={subItem?.['action']}
										>
											<div class="flex flex-row items-center gap-2">
												{#if subItem.icon}
													<svelte:component this={subItem.icon} size={16} />
												{/if}

												{subItem.label}
												{#if subItem?.['notificationCount']}
													<div class="ml-auto">
														<SideBarNotification notificationCount={subItem['notificationCount']} />
													</div>
												{/if}
											</div>
										</button>
									{:else}
										<a
											href={subItem.href}
											class="text-secondary font-normal block px-4 py-2 text-2xs hover:bg-surface-hover hover:text-primary"
											role="menuitem"
											tabindex="-1"
										>
											<div class="flex flex-row items-center gap-2">
												{#if subItem.icon}
													<svelte:component this={subItem.icon} size={16} />
												{/if}

												{subItem.label}

												{#if subItem?.['notificationCount']}
													<div class="ml-auto">
														<SideBarNotification notificationCount={subItem['notificationCount']} />
													</div>
												{/if}
											</div>
										</a>
									{/if}
								</div>
							</MenuItem>
						{/each}
					</Menu>
				{:else}
					<MenuLink class="!text-2xs" {...menuLink} {isCollapsed} />
				{/if}
			{/each}
		</div>
		<div class="space-y-0.5">
			{#each thirdMenuLinks as menuLink (menuLink)}
				{#if menuLink.subItems}
					<Menu>
						<div slot="trigger">
							<!-- svelte-ignore a11y-click-events-have-key-events -->
							<!-- svelte-ignore a11y-no-static-element-interactions -->
							<div
								class="relative"
								on:click={() => {
									if (menuLink.label === 'Help') {
										openChangelogs()
									}
								}}
							>
								<MenuButton class="!text-2xs" {...menuLink} {isCollapsed} />
								{#if menuLink.label === 'Help' && hasNewChangelogs}
									<span class="absolute top-1 right-1 flex h-2 w-2">
										<span
											class="animate-ping absolute inline-flex h-full w-full rounded-full bg-frost-400 opacity-75"
										/>
										<span class="relative inline-flex rounded-full h-2 w-2 bg-frost-500" />
									</span>
								{/if}
							</div>
						</div>
						{#each menuLink.subItems as subItem (subItem.href ?? subItem.label)}
							<MenuItem>
								<div class="py-1" role="none">
									<a
										href={subItem.href}
										class="text-secondary block px-4 py-2 text-xs hover:bg-surface-hover hover:text-primary relative"
										role="menuitem"
										tabindex="-1"
										target="_blank"
									>
										<div class="flex flex-row items-center gap-2">
											{#if subItem.icon}
												<svelte:component this={subItem.icon} size={16} />
											{/if}

											{subItem.label}
										</div>
									</a>
								</div>
							</MenuItem>
						{/each}
						{#if recentChangelogs.length > 0}
							<div class="w-full h-1 border-t" />
							<span class="text-xs px-4 font-bold"> Latest changelogs </span>
							{#each recentChangelogs as changelog}
								<MenuItem>
									<div class="py-1" role="none">
										<a
											href={changelog.href}
											class="text-secondary block px-4 py-2 text-xs hover:bg-surface-hover hover:text-primary relative"
											role="menuitem"
											tabindex="-1"
											target="_blank"
										>
											<div class="flex flex-row items-center gap-2">
												{changelog.label}
											</div>
										</a>
									</div>
								</MenuItem>
							{/each}
						{/if}
					</Menu>
				{/if}
			{/each}
		</div>
	</div>
</nav>

<ConfirmationModal
	open={leaveWorkspaceModal}
	title="Leave workspace"
	confirmationText="Remove"
	on:canceled={() => {
		leaveWorkspaceModal = false
	}}
	on:confirmed={() => {
		leaveWorkspace()
	}}
>
	<div class="flex flex-col w-full space-y-4">
		<span>Are you sure you want to leave this workspace?</span>
	</div>
</ConfirmationModal>
