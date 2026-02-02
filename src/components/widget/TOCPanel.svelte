<script lang="ts">
import I18nKey from "@i18n/i18nKey";
import { i18n } from "@i18n/translation";
import Icon from "@iconify/svelte";
import { onMount } from "svelte";

interface Heading {
    depth: number;
    slug: string;
    text: string;
}

let { headings: initialHeadings = [] }: { headings: Heading[] } = $props();

let isOpen = $state(false);
let panelEl: HTMLElement | undefined = $state(undefined);
let headings = $state<Heading[]>([...initialHeadings]);
let minDepth = $derived.by(() => {
    let min = 10;
    for (const heading of headings) {
        min = Math.min(min, heading.depth);
    }
    return min;
});

const removeTailingHash = (text: string) => {
    let lastIndexOfHash = text.lastIndexOf("#");
    if (lastIndexOfHash !== text.length - 1) {
        return text;
    }
    return text.substring(0, lastIndexOfHash);
};

function togglePanel() {
    isOpen = !isOpen;
}

function closePanel() {
    isOpen = false;
}

function handleClickOutside(event: MouseEvent) {
    if (panelEl && !panelEl.contains(event.target as Node)) {
        closePanel();
    }
}

function handleHeadingClick() {
    setTimeout(() => {
        closePanel();
    }, 100);
}

// Update headings from the desktop TOC component
function updateHeadingsFromDOM() {
    const tocWrapper = document.getElementById('toc');
    if (!tocWrapper) {
        headings = [];
        return;
    }
    
    const tocLinks = tocWrapper.querySelectorAll<HTMLAnchorElement>('a[href^="#"]');
    if (tocLinks.length === 0) {
        headings = [];
        return;
    }
    
    const newHeadings: Heading[] = [];
    
    tocLinks.forEach(link => {
        const hash = link.getAttribute('href')?.substring(1);
        if (!hash) return;
        
        const decodedHash = decodeURIComponent(hash);
        const targetElement = document.getElementById(decodedHash);
        if (!targetElement) return;
        
        const tagName = targetElement.tagName.toLowerCase();
        if (!tagName.match(/^h[1-6]$/)) return;
        
        const depth = parseInt(tagName.substring(1));
        newHeadings.push({
            depth: depth,
            slug: hash,
            text: targetElement.textContent || ''
        });
    });
    
    headings = newHeadings;
    isOpen = false;
}

function handleSwupContentReplace() {
    // Wait for DOM to be updated
    setTimeout(() => {
        updateHeadingsFromDOM();
    }, 150);
}

onMount(() => {
    // Set up click outside handler
    document.addEventListener("click", handleClickOutside);
    
    // Listen to Swup content replace events
    const setupSwupListener = () => {
        if (window.swup) {
            window.swup.hooks.on("content:replace", handleSwupContentReplace);
        }
    };
    
    if (window.swup) {
        setupSwupListener();
    } else {
        document.addEventListener("swup:enable", setupSwupListener);
    }
    
    // Cleanup
    return () => {
        document.removeEventListener("click", handleClickOutside);
        if (window.swup) {
            window.swup.hooks.off("content:replace", handleSwupContentReplace);
        }
    };
});
</script>

{#if headings.length > 0}
<div bind:this={panelEl} class="toc-panel-wrapper">
    <!-- Toggle Button -->
    <button
        onclick={togglePanel}
        class="toc-toggle-btn btn-card rounded-2xl shadow-lg"
        aria-label={i18n(I18nKey.toc)}
    >
        <Icon icon="material-symbols:toc-rounded" class="text-2xl text-[var(--primary)]"></Icon>
    </button>

    <!-- TOC Panel -->
    {#if isOpen}
    <div class="toc-mobile-panel card-base shadow-xl">
        <div class="toc-panel-header">
            <span class="font-bold text-lg">{i18n(I18nKey.toc)}</span>
            <button onclick={closePanel} class="btn-plain rounded-lg w-8 h-8" aria-label="Close">
                <Icon icon="material-symbols:close-rounded" class="text-lg"></Icon>
            </button>
        </div>
        <div class="toc-panel-content">
            {#each headings.filter((h) => h.depth < minDepth + 3) as heading, idx}
                <a
                    href={`#${heading.slug}`}
                    onclick={handleHeadingClick}
                    class="toc-item px-3 py-2 flex gap-2 relative transition w-full rounded-xl
                        hover:bg-[var(--toc-btn-hover)] active:bg-[var(--toc-btn-active)]"
                >
                    <div class="toc-badge transition w-5 h-5 shrink-0 rounded-lg text-xs flex items-center justify-center font-bold"
                        class:depth-0={heading.depth === minDepth}
                        class:depth-1={heading.depth === minDepth + 1}
                        class:depth-2={heading.depth === minDepth + 2}
                    >
                        {#if heading.depth === minDepth}
                            {headings.filter((h) => h.depth < minDepth + 3).slice(0, idx + 1).filter(h => h.depth === minDepth).length}
                        {:else if heading.depth === minDepth + 1}
                            <div class="transition w-2 h-2 rounded-[0.1875rem] bg-[var(--toc-badge-bg)]"></div>
                        {:else if heading.depth === minDepth + 2}
                            <div class="transition w-1.5 h-1.5 rounded-sm bg-black/5 dark:bg-white/10"></div>
                        {/if}
                    </div>
                    <div class="toc-text transition text-sm"
                        class:text-50={heading.depth === minDepth || heading.depth === minDepth + 1}
                        class:text-30={heading.depth === minDepth + 2}
                    >
                        {removeTailingHash(heading.text)}
                    </div>
                </a>
            {/each}
        </div>
    </div>
    {/if}
</div>
{/if}

<style lang="stylus">
.toc-panel-wrapper
    position: fixed
    bottom: 6rem
    right: 1rem
    z-index: 100
    @media (min-width: 1024px)
        right: 1.5rem

.toc-toggle-btn
    width: 3rem
    height: 3rem
    display: flex
    align-items: center
    justify-content: center
    @media (min-width: 1024px)
        width: 3.75rem
        height: 3.75rem

.toc-mobile-panel
    position: absolute
    bottom: 4rem
    right: 0
    width: 18rem
    max-height: 60vh
    border-radius: var(--radius-large)
    overflow: hidden
    display: flex
    flex-direction: column

.toc-panel-header
    display: flex
    justify-content: space-between
    align-items: center
    padding: 0.75rem 1rem
    border-bottom: 1px solid var(--line-divider)

.toc-panel-content
    overflow-y: auto
    padding: 0.5rem
    flex: 1

.toc-item
    .depth-0
        background: var(--toc-badge-bg)
        color: var(--btn-content)
    .depth-1
        margin-left: 1rem
    .depth-2
        margin-left: 2rem
</style>
