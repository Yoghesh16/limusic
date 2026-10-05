<script lang="ts">
	import { page } from '$app/state';
	import { scale } from 'svelte/transition';
	import { HugeiconsIcon } from '@hugeicons/svelte';
	import {
		Home01Icon,
		Search01Icon,
		LibraryIcon,
		Settings01Icon,
		Sun01Icon,
		Moon02Icon,
		Add01Icon,
		PinIcon,
		MusicNote01Icon,
		ListRestartIcon,
		ComputerIcon,
		SquareArrowLeft01Icon,
		SquareArrowRight01Icon
	} from '@hugeicons/core-free-icons';
	import { toggleMode } from 'mode-watcher';
	import { Button } from '$lib/components/ui/button';
	import { ON_REPEAT_ID, isLocalPlaylist, type BrowseItem } from '$lib/api';
	import { thumb } from '$lib/thumb';
	import PlaylistMenu from './PlaylistMenu.svelte';
	import { library, personal, ui, openNewPlaylist, toggleSidebar } from '$lib/player.svelte';
	import { mergeSaved, orderLibrary } from '$lib/personal';
	import { t } from '$lib/i18n.svelte';

	const nav = $derived([
		{ href: '/', label: t('nav.home'), icon: Home01Icon },
		{ href: '/search', label: t('nav.search'), icon: Search01Icon },
		{ href: '/library', label: t('nav.library'), icon: LibraryIcon }
	]);
	const isActive = (href: string) =>
		href === '/' ? page.url.pathname === '/' : page.url.pathname.startsWith(href);

	// Pinned first (in pin order), then everything else by last played. Derived here rather than in
	// the shared `library` store so the Library page keeps YouTube's own ordering. Playlists saved
	// on this machine sit in the same list: signed out they are the only ones there.
	const playlists = $derived(
		orderLibrary(mergeSaved(personal, library.items, 'playlist'), personal)
	);
	// How many of the leading rows are pinned — a rule under the last one explains the split.
	const pinnedCount = $derived(playlists.filter((p) => personal.pins.includes(p.id)).length);

	// YTM's library subtitle is "Owner • 20 tracks" and the rail is too narrow for both, so keep the
	// count and drop the rest. Subtitles without a number (albums: "Album • Artist") stay whole.
	const rowSubtitle = (s?: string) =>
		s
			?.split('•')
			.map((p) => p.trim())
			.filter((p) => /\d/.test(p))
			.at(-1) ?? s;

	const playlistHref = (item: BrowseItem) =>
		item.kind === 'album'
			? `/album/${encodeURIComponent(item.id)}`
			: item.kind === 'artist'
				? `/artist/${encodeURIComponent(item.id)}`
				: `/playlist/${encodeURIComponent(item.id)}`;

	// Account lives in the titlebar now — see AccountMenu.svelte.

	// Manual collapse is a large-screen preference: below lg the rail is already collapsed by the
	// breakpoint, so the button is hidden there and `wide()` has nothing to drop. Every expanded
	// style is an `lg:` class, so collapsing is just not emitting them. The flag lives in `ui`
	// because the overlays that offset by the sidebar's width read it too.
	const collapsed = $derived(ui.sidebarCollapsed);
	const wide = (cls: string) => (collapsed ? '' : cls);
</script>

<aside
	class="flex h-full w-16 shrink-0 flex-col border-r bg-sidebar p-3 text-sidebar-foreground {wide(
		'lg:w-60'
	)}"
>
	<div class="flex items-center justify-center px-2 py-2 {wide('lg:justify-between')}">
		<!-- music.youtube.com/img/on_platform_logo_dark.svg, wordmark switched to currentColor for light mode. -->
		<svg viewBox="0 0 77 26" class="hidden h-[26px] w-auto {wide('lg:block')}" fill="none" role="img" aria-label="YouTube Music">
			<g fill="currentColor">
				<path d="m30.112 21.8671h2.32v-7.29c0-2.04-.04-4.21-.2-7.10995h.26l.43 1.78 2.74 12.61995h2.36l2.69-12.61995.47-1.78h.24c-.12 2.56995-.19 4.88995-.19 7.10995v7.29h2.33v-16.79995h-3.96l-1.42 6.20995c-.6 2.58-1.03 5.78-1.28 7.41h-.19c-.18-1.66-.63-4.85-1.22-7.39l-1.46-6.22995h-3.92z" />
				<path d="m48.202 22.0571c1.46 0 2.37-.61 3.12-1.71h.11l.11 1.52h1.99v-12.35995h-2.64v9.92995c-.28.49-.93.85-1.54.85-.77 0-1.01-.61-1.01-1.63v-9.14995h-2.63v9.26995c0 2.01.58 3.28 2.49 3.28z" />
				<path d="m58.7536 22.1271c2.42 0 3.77-1.07 3.77-3.26 0-1.99-1-2.8-3.38-4.42-1.09-.72-1.68-1.17-1.68-2.23 0-.79.49-1.21 1.38-1.21.97 0 1.3.64 1.34 2.46l2.16-.12c.18-2.85-.84-4.06995-3.46-4.06995-2.48 0-3.67 1.06995-3.67 3.18995 0 1.96.93 2.85 2.68 4.07 1.55 1.07 2.4 1.74 2.4 2.63 0 .73-.51 1.26-1.4 1.26-1.02 0-1.6-.88-1.5-2.21l-2.19.04c-.34 2.56.86 3.87 3.55 3.87z" />
				<path d="m65.387 7.93715c.9 0 1.32-.3 1.32-1.54 0-1.16-.45-1.52-1.32-1.52-.88 0-1.31.32-1.31 1.52 0 1.24.41 1.54 1.31 1.54zm-1.22 13.92995h2.53v-12.35995h-2.53z" />
				<path d="m72.3428 22.0671c1.26 0 1.97-.1499 2.54-.69.85-.74 1.2-1.89 1.14-3.83l-2.31-.12c0 2.12-.34 2.92-1.33 2.92-1.09 0-1.27-1.19-1.27-3.35v-2.56c0-2.36.23-3.48 1.29-3.48.88 0 1.23.74 1.23 3.1l2.29-.16c.16-1.69-.01-3.07-.81-3.84-.6-.55995-1.49-.77995-2.67-.77995-3.03 0-3.97 1.91995-3.97 5.56995v1.69c0 3.65.71 5.53 3.87 5.53z" />
			</g>
			<path d="m13 26c7.176 0 13-5.824 13-13s-5.824-13-13-13-13 5.824-13 13 5.824 13 13 13z" fill="#f03" />
			<path d="m20.5 13c0 4.1439-3.3561 7.5-7.5 7.5-4.14386 0-7.5-3.3561-7.5-7.5 0-4.14386 3.35614-7.5 7.5-7.5 4.1439 0 7.5 3.35614 7.5 7.5z" stroke="#fff" />
			<path d="m17.75 13-7.5-4.25v8.5z" fill="#fff" />
		</svg>
		<!-- Column when collapsed: the two buttons don't fit side by side in the 64px rail. -->
		<div class="flex items-center gap-1 {collapsed ? 'flex-col' : ''}">
			<Button
				variant="ghost"
				size="icon-sm"
				class="hidden hover:text-primary lg:inline-flex"
				onclick={toggleSidebar}
				aria-label={collapsed ? t('a11y.expand_sidebar') : t('a11y.collapse_sidebar')}
			>
				<!-- altIcon/showAlt, not a ternary: `icon` is read once at mount. -->
				<HugeiconsIcon
					icon={SquareArrowLeft01Icon}
					altIcon={SquareArrowRight01Icon}
					showAlt={collapsed}
					strokeWidth={2}
					class="h-4 w-4"
				/>
			</Button>
			<Button
				variant="ghost"
				size="icon-sm"
				class="hover:text-primary"
				onclick={toggleMode}
				aria-label={t('a11y.toggle_theme')}
			>
				<HugeiconsIcon icon={Sun01Icon} strokeWidth={2} class="h-4 w-4 dark:hidden" />
				<HugeiconsIcon icon={Moon02Icon} strokeWidth={2} class="hidden h-4 w-4 dark:block" />
			</Button>
		</div>
	</div>

	<nav class="mt-2 flex flex-col gap-1">
		{#each nav as n (n.href)}
			<a
				href={n.href}
				title={n.label}
				class="group relative flex items-center justify-center gap-3 rounded-lg px-3 py-2 text-sm font-medium transition-colors {wide(
					'lg:justify-start'
				)} {isActive(n.href)
					? 'bg-primary/10 text-primary'
					: 'text-sidebar-foreground/70 hover:bg-sidebar-accent/50 hover:text-sidebar-foreground'}"
			>
				{#if isActive(n.href)}
					<span
						transition:scale={{ duration: 200, start: 0.4 }}
						class="absolute left-0 top-1/2 h-5 w-1 -translate-y-1/2 rounded-r-full bg-primary"
					></span>
				{/if}
				<HugeiconsIcon
					icon={n.icon}
					class="h-5 w-5 shrink-0"
				/>
				<span class="hidden {wide('lg:inline')}">{n.label}</span>
			</a>
		{/each}
		<button
			onclick={() => (ui.settingsOpen = true)}
			title={t('nav.settings')}
			class="group flex items-center justify-center gap-3 rounded-lg px-3 py-2 text-sm font-medium text-sidebar-foreground/70 transition-colors hover:bg-sidebar-accent/50 hover:text-sidebar-foreground {wide(
				'lg:justify-start'
			)}"
		>
			<HugeiconsIcon
				icon={Settings01Icon}
				class="h-5 w-5 shrink-0"
			/>
			<span class="hidden {wide('lg:inline')}">{t('nav.settings')}</span>
		</button>
	</nav>

	<!-- Playlists. Hidden on the icon rail (needs labels; matches YTM's collapsed rail). flex-1 lets
	     the list fill the space and scroll. Always there, signed out included: a playlist can be
	     made on this machine without an account (#251). -->
	<div class="mt-3 hidden min-h-0 flex-1 flex-col border-t pt-3 {wide('lg:flex')}">
		<Button variant="outline" size="sm" class="mb-2 w-full gap-2" onclick={() => openNewPlaylist()}>
			<HugeiconsIcon icon={Add01Icon} class="h-4 w-4" /> {t('nav.new_playlist')}
		</Button>
		<div class="min-h-0 flex-1 overflow-y-auto">
			{#each playlists as pl, i (pl.id)}
				<!-- The ⋯ is a sibling of the link, not a child: a <button> inside an <a> is invalid
				     HTML. pr-9 keeps the title clear of the button that overlays the row on hover. -->
				<div class="group/row relative" data-ctx>
					<a
						href={playlistHref(pl)}
						title={pl.title}
						class="flex items-center gap-2.5 rounded-lg py-1.5 pl-2 pr-9 transition-colors hover:bg-sidebar-accent/50"
					>
						<div
							class="relative h-10 w-10 shrink-0 overflow-hidden bg-muted {pl.kind === 'artist'
								? 'rounded-full'
								: 'rounded-md'}"
						>
							{#if pl.thumbnail && pl.id !== ON_REPEAT_ID}
								<img
									src={thumb(pl.thumbnail, 96)}
									alt=""
									class="h-full w-full object-cover"
									loading="lazy"
								/>
							{:else}
								<!-- On Repeat has no artwork by nature: icon tile, same as its card. -->
								<div
									class="flex h-full w-full items-center justify-center {pl.id === ON_REPEAT_ID
										? 'bg-primary/10 text-primary'
										: 'text-muted-foreground/50'}"
								>
									<!-- altIcon/showAlt, not a ternary: `icon` is read once at mount. -->
									<HugeiconsIcon
										icon={MusicNote01Icon}
										altIcon={ListRestartIcon}
										showAlt={pl.id === ON_REPEAT_ID}
										class={pl.id === ON_REPEAT_ID ? 'h-5 w-5' : 'h-4 w-4'}
									/>
								</div>
							{/if}
						</div>
						{#if personal.pins.includes(pl.id)}
							<span
								class="absolute left-9 top-0.5 flex h-4 w-4 items-center justify-center rounded-full bg-primary text-primary-foreground shadow"
							>
								<HugeiconsIcon icon={PinIcon} class="h-2.5 w-2.5" />
							</span>
						{/if}
						<div class="min-w-0 flex-1">
							<div class="truncate text-[13px] font-medium">{pl.title}</div>
							{#if pl.subtitle}
								<!-- The rail keeps only the count, so a playlist on this machine says
								     where it lives with an icon instead of the words. -->
								<div class="flex items-center gap-1 text-xs text-muted-foreground">
									{#if isLocalPlaylist(pl.id)}
										<HugeiconsIcon icon={ComputerIcon} class="h-3 w-3 shrink-0" />
									{/if}
									<span class="truncate">{rowSubtitle(pl.subtitle)}</span>
								</div>
							{/if}
						</div>
					</a>
					<PlaylistMenu item={pl} />
				</div>
				{#if pinnedCount && i === pinnedCount - 1}
					<div class="mx-3 my-1.5 h-px bg-border"></div>
				{/if}
			{:else}
				{#if library.loading}
					<p class="px-3 py-1.5 text-xs text-muted-foreground">{t('common.loading')}</p>
				{/if}
			{/each}
		</div>
	</div>
</aside>
