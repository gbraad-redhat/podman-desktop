<script lang="ts">
import { filtered, searchPattern, imagesInfos } from '../../stores/images';
import { onDestroy, onMount } from 'svelte';
import ImageEmptyScreen from './ImageEmptyScreen.svelte';
import FilteredEmptyScreen from '../ui/FilteredEmptyScreen.svelte';

import { router } from 'tinro';
import type { ImageInfoUI } from './ImageInfoUI';
import type { ImageInfo } from '../../../../main/src/plugin/api/image-info';
import NoContainerEngineEmptyScreen from './NoContainerEngineEmptyScreen.svelte';
import { providerInfos } from '../../stores/providers';
import { ImageUtils } from './image-utils';
import NavPage from '../ui/NavPage.svelte';
import ImageIcon from '../images/ImageIcon.svelte';
import type { Unsubscriber } from 'svelte/store';
import { containersInfos } from '../../stores/containers';
import type { ContainerInfo } from '../../../../main/src/plugin/api/container-info';
import moment from 'moment';
import Prune from '../engine/Prune.svelte';
import type { EngineInfoUI } from '../engine/EngineInfoUI';
import Button from '../ui/Button.svelte';
import { faArrowCircleDown, faCube, faTrash } from '@fortawesome/free-solid-svg-icons';
import Table from '../table/Table.svelte';
import { Column, Row } from '../table/table';
import ImageColumnStatus from './ImageColumnStatus.svelte';
import ImageColumnName from './ImageColumnName.svelte';
import ImageColumnEnvironment from './ImageColumnEnvironment.svelte';
import ImageColumnAge from './ImageColumnAge.svelte';
import ImageColumnSize from './ImageColumnSize.svelte';
import ImageColumnActions from './ImageColumnActions.svelte';

export let searchTerm = '';
$: searchPattern.set(searchTerm);

let images: ImageInfoUI[] = [];
let enginesList: EngineInfoUI[];

$: providerConnections = $providerInfos
  .map(provider => provider.containerConnections)
  .flat()
  .filter(providerContainerConnection => providerContainerConnection.status === 'started');

const imageUtils = new ImageUtils();

function updateImages() {
  const computedImages = storeImages
    .map((imageInfo: ImageInfo) => imageUtils.getImagesInfoUI(imageInfo, storeContainers))
    .flat();

  // update selected items based on current selected items
  computedImages.forEach(image => {
    const matchingImage = images.find(
      currentImage => currentImage.id === image.id && currentImage.engineId === image.engineId,
    );
    if (matchingImage) {
      image.selected = matchingImage.selected;
    }
  });
  computedImages.sort((first, second) => second.createdAt - first.createdAt);
  images = computedImages;

  // Map engineName, engineId and engineType from currentContainers to EngineInfoUI[]
  const engines = images.map(container => {
    return {
      name: container.engineName,
      id: container.engineId,
    };
  });

  // Remove duplicates from engines by name
  const uniqueEngines = engines.filter((engine, index, self) => index === self.findIndex(t => t.name === engine.name));

  // Set the engines to the global variable for the Prune functionality button
  enginesList = uniqueEngines;

  // compute refresh interval
  const interval = computeInterval();
  refreshTimeouts.push(setTimeout(refreshAge, interval));
}

let imagesUnsubscribe: Unsubscriber;
let containersUnsubscribe: Unsubscriber;
let storeContainers: ContainerInfo[] = [];
let storeImages: ImageInfo[] = [];

onMount(async () => {
  containersUnsubscribe = containersInfos.subscribe(value => {
    storeContainers = value;
    updateImages();
  });

  imagesUnsubscribe = filtered.subscribe(value => {
    storeImages = value;
    updateImages();
  });
});

onDestroy(() => {
  // kill timers
  refreshTimeouts.forEach(timeout => clearTimeout(timeout));
  refreshTimeouts.length = 0;

  // unsubscribe from the store
  if (imagesUnsubscribe) {
    imagesUnsubscribe();
  }
  if (containersUnsubscribe) {
    containersUnsubscribe();
  }
});

function gotoBuildImage(): void {
  router.goto('/images/build');
}

function gotoPullImage(): void {
  router.goto('/images/pull');
}

// delete the items selected in the list
let bulkDeleteInProgress = false;
async function deleteSelectedImages() {
  bulkDeleteInProgress = true;
  const selectedImages = images.filter(image => image.selected);
  await selectedImages.reduce((prev: Promise<void>, image) => {
    return prev
      .then(() => imageUtils.deleteImage(image))
      .catch((e: unknown) => console.log('error while removing image', e));
  }, Promise.resolve());
  bulkDeleteInProgress = false;
}

let refreshTimeouts: NodeJS.Timeout[] = [];
const SECOND = 1000;
function refreshAge() {
  images = images.map(imageInfo => {
    return { ...imageInfo, age: imageUtils.refreshAge(imageInfo) };
  });

  // compute new interval
  const newInterval = computeInterval();
  refreshTimeouts.forEach(timeout => clearTimeout(timeout));
  refreshTimeouts.length = 0;
  refreshTimeouts.push(setTimeout(refreshAge, newInterval));
}

function computeInterval(): number {
  // no images, no refresh
  if (images.length === 0) {
    return -1;
  }

  // do we have images that have been created in less than 1 minute
  // if so, need to update every second
  const imagesCreatedInLessThan1Mn = images.filter(image => moment().diff(moment.unix(image.createdAt), 'minutes') < 1);
  if (imagesCreatedInLessThan1Mn.length > 0) {
    return 2 * SECOND;
  }

  // every minute for images created less than 1 hour
  const imagesCreatedInLessThan1Hour = images.filter(image => moment().diff(moment.unix(image.createdAt), 'hours') < 1);
  if (imagesCreatedInLessThan1Hour.length > 0) {
    // every minute
    return 60 * SECOND;
  }

  // every hour for images created less than 1 day
  const imagesCreatedInLessThan1Day = images.filter(image => moment().diff(moment.unix(image.createdAt), 'days') < 1);
  if (imagesCreatedInLessThan1Day.length > 0) {
    // every hour
    return 60 * 60 * SECOND;
  }

  // every day
  return 60 * 60 * 24 * SECOND;
}

let selectedItemsNumber: number;
let table: Table;

let statusColumn = new Column<ImageInfoUI>('Status', {
  align: 'center',
  width: '70px',
  renderer: ImageColumnStatus,
  comparator: (a, b) => Number(b.inUse) - Number(a.inUse),
});

let nameColumn = new Column<ImageInfoUI>('Name', {
  width: '4fr',
  renderer: ImageColumnName,
  comparator: (a, b) => a.name.localeCompare(b.name),
});

let envColumn = new Column<ImageInfoUI>('Environment', {
  renderer: ImageColumnEnvironment,
  comparator: (a, b) => a.engineName.localeCompare(b.engineName),
});

let ageColumn = new Column<ImageInfoUI>('Age', {
  renderer: ImageColumnAge,
  comparator: (a, b) => moment().diff(moment.unix(a.createdAt)) - moment().diff(moment.unix(b.createdAt)),
});

let sizeColumn = new Column<ImageInfoUI>('Size', {
  align: 'right',
  renderer: ImageColumnSize,
  comparator: (a, b) => b.size - a.size,
});

const columns: Column<ImageInfoUI>[] = [
  statusColumn,
  nameColumn,
  envColumn,
  ageColumn,
  sizeColumn,
  new Column<ImageInfoUI>('Actions', { align: 'right', width: '150px', renderer: ImageColumnActions }),
];

const row = new Row<ImageInfoUI>({ selectable: image => !image.inUse, disabledText: 'Image is used by a container' });
</script>

<NavPage bind:searchTerm="{searchTerm}" title="images">
  <svelte:fragment slot="additional-actions">
    {#if $imagesInfos.length > 0}
      <Prune type="images" engines="{enginesList}" />
    {/if}
    <Button on:click="{() => gotoPullImage()}" title="Pull Image From a Registry" icon="{faArrowCircleDown}">
      Pull
    </Button>
    <Button on:click="{() => gotoBuildImage()}" title="Build Image from Containerfile" icon="{faCube}">Build</Button>
  </svelte:fragment>

  <svelte:fragment slot="bottom-additional-actions">
    {#if selectedItemsNumber > 0}
      <Button
        on:click="{() => deleteSelectedImages()}"
        title="Delete {selectedItemsNumber} selected items"
        bind:inProgress="{bulkDeleteInProgress}"
        icon="{faTrash}" />
      <span>On {selectedItemsNumber} selected items.</span>
    {/if}
  </svelte:fragment>

  <div class="flex min-w-full h-full" slot="content">
    <Table
      kind="image"
      bind:this="{table}"
      bind:selectedItemsNumber="{selectedItemsNumber}"
      data="{images}"
      columns="{columns}"
      row="{row}"
      defaultSortColumn="Age">
    </Table>

    {#if providerConnections.length === 0}
      <NoContainerEngineEmptyScreen />
    {:else if $filtered.length === 0}
      {#if searchTerm}
        <FilteredEmptyScreen icon="{ImageIcon}" kind="images" bind:searchTerm="{searchTerm}" />
      {:else}
        <ImageEmptyScreen />
      {/if}
    {/if}
  </div>
</NavPage>
