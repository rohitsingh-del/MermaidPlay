<script lang="ts">
  import Card from '$/components/Card/Card.svelte';
  import CopyButton from '$/components/CopyButton.svelte';
  import { Button } from '$/components/ui/button';
  import { Input } from '$/components/ui/input';
  import { Separator } from '$/components/ui/separator';
  import * as ToggleGroup from '$/components/ui/toggle-group';
  import { browser } from '$app/environment';
  import { waitForRender } from '$lib/util/autoSync';
  import { inputStateStore, stateStore, urlsStore } from '$lib/util/state';
  import { logEvent } from '$lib/util/stats';
  import dayjs from 'dayjs';
  import { toBase64 } from 'js-base64';
  import DownloadIcon from '~icons/material-symbols/download';
  import WidthIcon from '~icons/material-symbols/width-rounded';

  type Exporter = (context: CanvasRenderingContext2D, image: HTMLImageElement) => () => void;

  const getFileName = (extension: string) =>
    `mermaid-diagram-${dayjs().format('YYYY-MM-DD-HHmmss')}.${extension}`;

  /**
   * Fix text clipping in exported SVG for hand-drawn (rough) mode.
   * svg2roughjs copies foreignObject elements but their height is often insufficient,
   * causing text bottom edges to be cut off regardless of language.
   */
  const fixForeignObjectClipping = (svg: HTMLElement) => {
    const foreignObjects = svg.querySelectorAll('foreignObject');
    foreignObjects.forEach((foreignObj) => {
      const currentHeight = parseFloat(foreignObj.getAttribute('height') || '0');
      if (currentHeight <= 0) return;

      const currentY = parseFloat(foreignObj.getAttribute('y') || '0');
      const newHeight = currentHeight * 1.5;
      const heightDiff = newHeight - currentHeight;

      foreignObj.setAttribute('height', newHeight.toString());
      foreignObj.setAttribute('y', (currentY - heightDiff / 2).toString());

      // Ensure inner HTML elements are vertically centered within the expanded area
      const htmlElements = foreignObj.querySelectorAll('div, span, p');
      htmlElements.forEach((htmlEl) => {
        const el = htmlEl as HTMLElement;
        el.style.display = 'flex';
        el.style.alignItems = 'center';
        el.style.justifyContent = 'center';
        el.style.height = '100%';
      });
    });
  };

  const getSvgElement = () => {
    const svgElement = document.querySelector('#container svg')?.cloneNode(true) as HTMLElement;
    svgElement.setAttribute('xmlns:xlink', 'http://www.w3.org/1999/xlink');
    return svgElement;
  };

  const getBase64SVG = (svg?: HTMLElement, width?: number, height?: number): string => {
    if (svg) {
      // Prevents the SVG size of the interface from being changed
      svg = svg.cloneNode(true) as HTMLElement;
    }
    if (height) {
      svg?.setAttribute('height', `${height}px`);
    }
    if (width) {
      svg?.setAttribute('width', `${width}px`);
    }
    // Workaround https://stackoverflow.com/questions/28690643/firefox-error-rendering-an-svg-image-to-html5-canvas-with-drawimage

    if (!svg) {
      svg = getSvgElement();
    }

    if ($stateStore.rough) {
      fixForeignObjectClipping(svg);
    }

    svg.style.backgroundColor = window
      .getComputedStyle(document.body)
      .getPropertyValue('--background');

    const svgString = svg.outerHTML
      .replaceAll('<br>', '<br/>')
      .replaceAll(/<img([^>]*)>/g, (m, g: string) => `<img ${g} />`);

    return toBase64(`<?xml version="1.0" encoding="UTF-8"?>
${svgString}`);
  };

  const simulateDownload = (download: string, href: string): void => {
    const a = document.createElement('a');
    a.download = download;
    a.href = href;
    a.click();
    a.remove();
  };

  const exportImage = async (event: Event, exporter: Exporter) => {
    $inputStateStore.panZoom = false;
    await new Promise((resolve) => setTimeout(resolve, 1000));
    await waitForRender();
    const canvas = document.createElement('canvas');
    const svg = document.querySelector<HTMLElement>('#container svg');
    if (!svg) {
      throw new Error('svg not found');
    }

    const box = svg.getBoundingClientRect();

    // In rough mode, SVG has width/height="100%" so getBoundingClientRect returns
    // the container size, not the actual diagram size. Use viewBox dimensions instead.
    const svgEl = svg as unknown as SVGSVGElement;
    const viewBox = svgEl.viewBox?.baseVal;
    const contentWidth = viewBox && viewBox.width > 0 ? viewBox.width : box.width;
    const contentHeight = viewBox && viewBox.height > 0 ? viewBox.height : box.height;

    if (imageSizeMode === 'width') {
      const ratio = contentHeight / contentWidth;
      canvas.width = imageSize;
      canvas.height = imageSize * ratio;
    } else if (imageSizeMode === 'height') {
      const ratio = contentWidth / contentHeight;
      canvas.width = imageSize * ratio;
      canvas.height = imageSize;
    } else {
      const multiplier = 2;
      canvas.width = contentWidth * multiplier;
      canvas.height = contentHeight * multiplier;
    }

    const context = canvas.getContext('2d');
    if (!context) {
      throw new Error('context not found');
    }

    context.fillStyle = window.getComputedStyle(document.body).getPropertyValue('--background');
    context.fillRect(0, 0, canvas.width, canvas.height);

    const image = new Image();
    image.addEventListener('load', () => {
      exporter(context, image)();
      $inputStateStore.panZoom = true;
    });
    image.src = `data:image/svg+xml;base64,${getBase64SVG(svg, canvas.width, canvas.height)}`;
    // Fallback to set panZoom to true after 2 seconds
    // This is a workaround for the case when the image is not loaded
    setTimeout(() => {
      if (!$inputStateStore.panZoom) {
        $inputStateStore.panZoom = true;
      }
    }, 2000);
    event.stopPropagation();
    event.preventDefault();
  };

  const downloadImage: Exporter = (context, image) => {
    return () => {
      const { canvas } = context;
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      simulateDownload(
        getFileName('png'),
        canvas.toDataURL('image/png').replace('image/png', 'image/octet-stream')
      );
    };
  };

  const isClipboardAvailable = (): boolean => {
    return Object.prototype.hasOwnProperty.call(window, 'ClipboardItem');
  };

  const clipboardCopy: Exporter = (context, image) => {
    return () => {
      const { canvas } = context;
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      canvas.toBlob((blob) => {
        try {
          if (!blob) {
            throw new Error('blob is empty');
          }
          void navigator.clipboard.write([
            new ClipboardItem({
              [blob.type]: blob
            })
          ]);
        } catch (error) {
          console.error(error);
        }
      });
    };
  };

  const onCopyClipboard = async (event: Event) => {
    await exportImage(event, clipboardCopy);
    logEvent('copyClipboard');
  };

  const onDownloadPNG = async (event: Event) => {
    await exportImage(event, downloadImage);
    logEvent('download', {
      type: 'png'
    });
  };

  const onDownloadSVG = () => {
    simulateDownload(getFileName('svg'), `data:image/svg+xml;base64,${getBase64SVG()}`);
    logEvent('download', {
      type: 'svg'
    });
  };

  const downloadJPG: Exporter = (context, image) => {
    return () => {
      const { canvas } = context;
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      simulateDownload(
        getFileName('jpg'),
        canvas.toDataURL('image/jpeg', 0.95).replace('image/jpeg', 'image/octet-stream')
      );
    };
  };

  const onDownloadJPG = async (event: Event) => {
    await exportImage(event, downloadJPG);
    logEvent('download', {
      type: 'jpg'
    });
  };

  const onDownloadPDF = async (event: Event) => {
    $inputStateStore.panZoom = false;
    await new Promise((resolve) => setTimeout(resolve, 1000));
    await waitForRender();
    const svg = document.querySelector<HTMLElement>('#container svg');
    if (!svg) {
      throw new Error('svg not found');
    }
    const box = svg.getBoundingClientRect();
    const svgEl = svg as unknown as SVGSVGElement;
    const viewBox = svgEl.viewBox?.baseVal;
    const contentWidth = viewBox && viewBox.width > 0 ? viewBox.width : box.width;
    const contentHeight = viewBox && viewBox.height > 0 ? viewBox.height : box.height;

    const canvas = document.createElement('canvas');
    const multiplier = 2;
    canvas.width = contentWidth * multiplier;
    canvas.height = contentHeight * multiplier;
    const context = canvas.getContext('2d');
    if (!context) {
      throw new Error('context not found');
    }
    context.fillStyle = window.getComputedStyle(document.body).getPropertyValue('--background');
    context.fillRect(0, 0, canvas.width, canvas.height);

    const image = new Image();
    image.addEventListener('load', () => {
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      // Create a simple PDF with embedded image
      const imgData = canvas.toDataURL('image/jpeg', 0.95);
      // Use a minimal PDF approach: open in new window for print/save
      const printWindow = window.open('');
      if (printWindow) {
        printWindow.document.write(`<html><head><title>Mermaid Play Diagram</title></head><body style="margin:0;display:flex;justify-content:center;align-items:center;min-height:100vh;background:#fff;"><img src="${imgData}" style="max-width:100%;max-height:100vh;" /></body></html>`);
        printWindow.document.close();
        printWindow.focus();
        printWindow.print();
      }
      $inputStateStore.panZoom = true;
    });
    image.src = `data:image/svg+xml;base64,${getBase64SVG(svg, canvas.width, canvas.height)}`;
    setTimeout(() => {
      if (!$inputStateStore.panZoom) {
        $inputStateStore.panZoom = true;
      }
    }, 2000);
    event.stopPropagation();
    event.preventDefault();
    logEvent('download', { type: 'pdf' });
  };

  let imageSizeMode: 'auto' | 'width' | 'height' = $state('auto');

  $effect(() => {
    if (!imageSizeMode) {
      imageSizeMode = 'auto';
    }
  });

  let imageSize = $state(1080);
</script>

{#snippet dualActionButton(text: string, download: (event: Event) => unknown)}
  <div class="flex flex-grow gap-0.5">
    <Button
      class="flex-grow"
      onclick={download}
      data-testid="download-{text}">
      <DownloadIcon />
      {text}
    </Button>
  </div>
{/snippet}

<Card title="Actions" isStackable icon={{ component: DownloadIcon, class: 'rotate-180' }}>
  <div class="flex min-w-fit flex-col gap-2 p-2">
    <div class="flex w-full items-center gap-2 py-2 whitespace-nowrap">
      PNG size
      <ToggleGroup.Root type="single" variant="outline" bind:value={imageSizeMode} aria-label="Image Size Mode">
        <ToggleGroup.Item value="auto">Auto</ToggleGroup.Item>
        <ToggleGroup.Item value="width">Width</ToggleGroup.Item>
        <ToggleGroup.Item value="height">Height</ToggleGroup.Item>
      </ToggleGroup.Root>
      {#if imageSizeMode !== 'auto'}
        <WidthIcon
          class={['size-6 shrink-0 transition-all', imageSizeMode === 'width' && 'rotate-90']} />
      {/if}
      <Input
        type="number"
        min="3"
        max="10000"
        disabled={imageSizeMode === 'auto'}
        bind:value={imageSize}
        aria-label="Image Size Value" />
    </div>
    <div class="flex gap-2">
      {@render dualActionButton('PNG', onDownloadPNG)}
      {@render dualActionButton('SVG', onDownloadSVG)}
    </div>
    <div class="flex gap-2">
      {@render dualActionButton('JPG', onDownloadJPG)}
      {@render dualActionButton('PDF', onDownloadPDF)}
    </div>
    <Separator />
    {#if isClipboardAvailable()}
      <CopyButton onclick={onCopyClipboard} label="Copy Image" />
    {/if}
  </div>
</Card>
