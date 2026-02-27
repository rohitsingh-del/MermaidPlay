<script lang="ts" module>
  import { logEvent, plausible } from '$lib/util/stats';
  import { version } from 'mermaid/package.json';

  void logEvent('version', {
    mermaidVersion: version
  });
</script>

<script lang="ts">
  import MainMenu from '$/components/MainMenu.svelte';
  import { Button } from '$/components/ui/button';
  import { Separator } from '$/components/ui/separator';
  import { dismissPromotion, getActivePromotion } from '$lib/util/promos/promo';
  import type { ComponentProps, Snippet } from 'svelte';
  import PlayIcon from '~icons/material-symbols/play-circle-outline-rounded';
  import CloseIcon from '~icons/material-symbols/close-rounded';
  import GithubIcon from '~icons/mdi/github';
  import DropdownNavMenu from './DropdownNavMenu.svelte';

  interface Props {
    mobileToggle?: Snippet;
    children: Snippet;
  }

  let { children, mobileToggle }: Props = $props();



  let activePromotion = $state(getActivePromotion());

  const trackBannerClick = () => {
    if (!plausible || !activePromotion) {
      return;
    }
    logEvent('bannerClick', {
      promotion: activePromotion.id
    });
  };
</script>

{#if activePromotion}
  <div class="top-bar z-10 flex h-fit w-full bg-primary">
    <div
      class="flex flex-grow"
      role="button"
      tabindex="0"
      onclick={trackBannerClick}
      onkeypress={trackBannerClick}>
      <activePromotion.component {closeBanner} />
    </div>
    {#snippet closeBanner()}
      <Button
        title="Dismiss banner"
        aria-label="Dismiss banner"
        variant="ghost"
        class="hover:bg-transparent hover:text-[#261A56]"
        size="sm"
        onclick={() => {
          dismissPromotion(activePromotion?.id);
          activePromotion = undefined;
        }}>
        <CloseIcon />
      </Button>
    {/snippet}
  </div>
{/if}

<nav class="z-50 flex p-4 sm:p-6">
  <div class="flex flex-1 items-center gap-2">
    <MainMenu />
    <PlayIcon class="size-6 text-accent" />
    <a href="/" class="whitespace-nowrap text-accent" aria-label="Mermaid Play Home">
      {#if !mobileToggle}
        Mermaid
      {/if}
      Play
    </a>
  </div>
  <div
    id="menu"
    class="hidden flex-nowrap items-center justify-between gap-3 overflow-hidden md:flex">
    {@render children()}
  </div>
  {@render mobileToggle?.()}
</nav>
