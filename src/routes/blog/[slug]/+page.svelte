<script lang="ts">
	import { onDestroy, afterUpdate, tick } from 'svelte';
	import Support from '$lib/Support.svelte';
	import { humanDate } from '$lib/formatters';
	import Head from '$lib/Head.svelte';
	import Comments from '../../../lib/Comments.svelte';
	import { blog } from '$lib/current-blog.store';

	/** @type {import('./$types').PageData} */
	export let data;
	const { post } = data;

	const logos = post.metadata.tags
		.map((tag) => {
			switch (tag.toLowerCase()) {
				case '.net':
				case 'dotnet':
					return { src: 'dotnet.svg', alt: 'The .NET logo' };
				case 'angular':
					return { src: 'angular.svg', alt: 'The Angular logo' };
				case 'playwright':
					return { src: 'playwright.svg', alt: 'The Playwright logo' };
				case 'ngrx':
					return { src: 'ngrx.svg', alt: 'The NgRx logo' };
				case 'azure':
					return { src: 'azure.svg', alt: 'The Azure logo' };
				case 'zod':
					return { src: 'zod.svg', alt: 'The zod logo' };
				case 'angular testing library':
					return { src: 'atl.png', alt: 'The Angular Testing Library logo' };
				default:
					return null;
			}
		})
		.filter(Boolean);

	let scrollY;

	let pres: HTMLElement[] = [];
	let copyButtons: HTMLElement[] = [];

	onDestroy(() => {
		unRegisterCopyClick();
		blog.reset();
	});

	let headings = null;
	afterUpdate(async () => {
		const hasTldr = post.tldr && new URLSearchParams(window.location.search).get('tldr') !== null;
		blog.loadBlog(post.metadata.title, hasTldr ? 'tldr' : post.tldr ? 'detailed' : 'single');
		headings = hasTldr
			? null
			: headings || window.history.pushState
			? [...(document.querySelectorAll('main h2,h3') as any)].reverse()
			: [];

		await tick();
		unRegisterCopyClick();
		pres = [...(document.querySelectorAll('pre') as any)];
		pres.forEach((pre) => pre.addEventListener('click', copyLinkToCodeBlock));

		copyButtons = [...(document.querySelectorAll('.copy-code') as any)];
		copyButtons.forEach((pre) => pre.addEventListener('click', copyCodeOnClick));
	});

	let lastHeading = null;
	$: {
		if (typeof window !== 'undefined') {
			if ($blog?.state === 'tldr' && lastHeading) {
				lastHeading = null;
				window.history.replaceState(window.history.state, '', ' ');
			} else if ($blog?.state !== 'tldr' && headings) {
				const heading = headings.find((h) => h.offsetTop <= scrollY);
				if (lastHeading !== heading) {
					lastHeading = heading;
					window.history.replaceState(window.history.state, '', heading ? `#${heading?.id}` : ' ');
				}
			}
		}
	}

	function unRegisterCopyClick() {
		if (typeof document !== 'undefined') {
			pres.forEach((pre) => pre.removeEventListener('click', copyLinkToCodeBlock));
			copyButtons.forEach((pre) => pre.removeEventListener('click', copyCodeOnClick));
		}
	}

	function copyLinkToCodeBlock(e: PointerEvent) {
		if (e.ctrlKey && navigator.clipboard && navigator.clipboard.writeText) {
			const { origin, pathname } = window.location;
			navigator.clipboard.writeText(`${origin}${pathname}#${(e.currentTarget as HTMLElement).id}`);
		}
	}

	function copyCodeOnClick(e: PointerEvent) {
		if (e.target instanceof HTMLElement) {
			const target = e.target;
			const ref = target.getAttribute('data-ref');
			const code = document.querySelector(`[id="${ref}"] code`);
			if (code instanceof HTMLElement) {
				navigator.clipboard.writeText(code.innerText).then(() => {
					target.innerHTML = 'assignment_turned_in';
					target.classList.add('success');
					setTimeout(() => {
						target.classList.remove('success');
						target.innerHTML = 'content_paste';
					}, 1000);
				});
			}
		}
	}
</script>

<Head title="{post.metadata.title} - Tim Deschryver" details={false} />

<svelte:head>
	<link rel="canonical" href={post.metadata.canonical} />

	<meta name="author" content={post.metadata.author} />
	<meta name="copyright" content={post.metadata.author} />
	<meta name="title" content={post.metadata.title} />
	<meta name="description" content={post.metadata.description} />
	<meta name="keywords" content={post.metadata.tags.join(',')} />
	<meta name="image" content={post.metadata.banner} />

	<meta name="twitter:card" content="summary_large_image" />
	<meta name="twitter:image" content={post.metadata.banner} />
	<meta name="twitter:image:alt" content={post.metadata.title} />
	<meta name="twitter:title" content={post.metadata.title} />
	<meta name="twitter:description" content={post.metadata.description} />
	<meta name="twitter:label1" content="Written by" />
	<meta name="twitter:data1" content={post.metadata.author} />
	<meta name="twitter:label2" content="Published on" />
	<meta name="twitter:data2" content={humanDate(post.metadata.date)} />

	<meta name="og:url" content={post.metadata.canonical} />
	<meta name="og:title" content={post.metadata.title} />
	<meta name="og:description" content={post.metadata.description} />
	<meta name="og:type" content="article" />
	<meta name="og:image" content={post.metadata.banner} />
</svelte:head>

<svelte:window bind:scrollY />

<header>
	<h1>{post.metadata.title}</h1>
	<img class="banner" src={post.metadata.banner} alt={post.metadata.title} />
	<div class="details">
		<div class="published-at">
			{#if post.metadata.modified !== post.metadata.date}
				<time datetime={humanDate(post.metadata.modified)}>
					Modified {humanDate(post.metadata.modified)}</time
				>
			{:else}
				<time datetime={humanDate(post.metadata.date)}
					>Published {humanDate(post.metadata.date)}</time
				>
			{/if}
		</div>

		<div class="logos">
			{#each logos as logo}
				<img class="mt-0 logo" src="/images/{logo.src}" alt={logo.alt} />
			{/each}
		</div>

		<div class="mt-0 author">
			<img class="author-img" src="/images/tim.jpg" alt="profile" />
			<div class="mt-0">
				<div class="author-name">Tim Deschryver</div>
				<div class="author-source mt-0">timdeschryver.dev</div>
			</div>
		</div>
	</div>
</header>

<div class="side-actions" hidden={(scrollY || 0) < 1000}>
	{#if post.metadata.translations}
		<div>Translations</div>
		{#each post.metadata.translations as translation}
			<a href={translation.url} rel="external">{translation.language}</a>
		{/each}
	{/if}
</div>

{#if post.metadata.translations}
	<div class="translations">
		<hr />
		<p>This post is also available in:</p>
		<ul>
			{#each post.metadata.translations as translation}
				<li>
					<a href={translation.url} rel="external">{translation.language}</a> by
					<a href={translation.profile} rel="external">{translation.author}</a>
				</li>
			{/each}
		</ul>
		<hr />
	</div>
{/if}

{#if post.tldr}
	<button class="tldr" on:click={blog.toggleTldr}>
		👀 {$blog?.state === 'tldr'
			? 'I want to read the blog post'
			: 'Just show me the code already'}</button
	>
{/if}

{#if $blog?.state === 'tldr' && post.tldr}
	{@html post.tldr}
{:else}
	{@html post.html}

	{#if post.contributors.length}
		<h5>A warm thank you to the contributors of this blog post</h5>
		<ul class="mt-0">
			{#each post.contributors as [login, name]}
				<li>
					<a href={`https://github.com/${login}`} rel="external">{name}</a>
				</li>
			{/each}
		</ul>
	{/if}

	{#if post.metadata.incomingLinks.length}
		<h5>Incoming links</h5>
		<ul class="mt-0" data-sveltekit-reload>
			{#each post.metadata.incomingLinks as link}
				<li>
					<a href={`/blog/${link.slug}`}>{link.title}</a>
				</li>
			{/each}
		</ul>
	{/if}

	{#if post.metadata.outgoingLinks.length}
		<h5>Outgoing links</h5>
		<ul class="mt-0" data-sveltekit-reload>
			{#each post.metadata.outgoingLinks as link}
				<li>
					<a href={`/blog/${link.slug}`}>{link.title}</a>
				</li>
			{/each}
		</ul>
	{/if}
{/if}

<Support />

<Comments />

<div class="article-actions">
	<a class="mt-0" target="_blank" rel="noreferrer" href="https://timdeschryver.dev/support">
		Support the blog
	</a>
	<a
		class="mt-0"
		target="_blank"
		rel="noreferrer"
		href="https://twitter.com/intent/tweet?text={post.metadata.title}&via=tim_deschryver&url={post
			.metadata.canonical}"
	>
		Share on Twitter
	</a>
	<a
		class="mt-0"
		href="https://twitter.com/search?q={post.metadata.canonical}"
		target="_blank"
		rel="noreferrer"
	>
		Discuss on Twitter
	</a>
	<a class="mt-0" target="_blank" rel="noreferrer" href={post.metadata.edit}> Edit on GitHub </a>
</div>

<style>
	.tldr {
		background: none;
		border: none;
		text-align: center;
		font-weight: 900;
		margin-top: var(--spacing);
	}

	.side-actions {
		display: block;
		position: fixed;
		margin-top: var(--header-height);
		top: 20px;
		left: 20px;
	}

	.side-actions div {
		border: none;
		margin-top: var(--spacing);
	}

	.side-actions > * {
		padding: 4px;
		display: block;
		text-align: center;
		border: 1px solid;
		background: none;
		width: 100%;
		cursor: pointer;
		color: var(--text-color-light);
		margin-top: 3px;
		margin-bottom: 0;
	}

	:global(#svelte > main) > header {
		grid-column: 1 / 4;
		min-height: 100vh;
		min-width: 90%;
		max-width: 90%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: space-evenly;
		text-align: center;
		margin: 0 auto;
		padding: 0;
		font-size: clamp(1rem, 3vw, 6rem);
	}

	.banner {
		display: none;
	}

	.details {
		display: flex;
		justify-content: space-between;
		margin: 0;
		font-size: 1.5rem;
		width: 100%;
		align-items: center;
	}

	.author-img,
	.logo {
		width: 64px;
		height: 64px;
	}

	.author {
		display: flex;
		align-items: center;
		text-align: left;
		gap: 0.5em;
	}

	.author-name {
		font-size: 1.3rem;
	}
	.author-source {
		font-size: 0.8rem;
		transform: translate(3px, -8px);
		display: none;
	}

	.author-img {
		width: auto;
		border-radius: 100%;
	}

	.details > * {
		margin-top: 0;
	}

	.logos {
		display: flex;
		gap: 0.6em;
		display: none;
	}

	@media screen and (max-width: 1150px) {
		.details {
			font-size: 1rem;
		}

		.author-name {
			font-size: 1rem;
		}

		.author-img,
		.logo {
			width: 48px;
			height: 48px;
		}
	}

	@media (max-width: 1450px) {
		.side-actions {
			width: 130px;
		}
	}

	@media (max-width: 1380px) {
		.side-actions {
			display: none;
		}
	}

	.article-actions {
		display: flex;
		justify-content: space-evenly;
	}

	.translations ul {
		list-style: none;
		margin-top: var(--spacing-small);
	}
</style>
