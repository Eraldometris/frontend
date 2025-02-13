<!--
  Global contextmenu for a (media) item.
  Because this dialog can be called from various places throughout the app,
  we steer its visibility through the centralized eventbus.
-->
<template>
  <v-menu
    v-model="show"
    :target="[posX, posY]"
    scrim
    @update:model-value="
      (v) => {
        store.dialogActive = v;
      }
    "
  >
    <v-card min-width="260">
      <v-list density="compact" slim tile>
        <div v-for="menuItem of items" :key="menuItem.label" class="menurow">
          <v-list-item
            variant="text"
            :title="$t(menuItem.label, menuItem.labelArgs || [])"
            :disabled="menuItem.disabled == true"
            :prepend-icon="menuItem.icon"
            :append-icon="menuItem.selected ? 'mdi-check' : undefined"
            @click="() => (menuItem.action ? menuItem.action() : '')"
          />
        </div>
      </v-list>
    </v-card>
  </v-menu>
</template>

<script setup lang="ts">
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue';
import api from '@/plugins/api';
import { store } from '@/plugins/store';
import { ItemContextMenuDialogEvent, eventbus } from '@/plugins/eventbus';

const show = ref<boolean>(false);
const items = ref<ContextMenuItem[]>([]);
const posX = ref(0);
const posY = ref(0);

onMounted(() => {
  eventbus.on('contextmenu', async (evt: ItemContextMenuDialogEvent) => {
    items.value = evt.items;
    posX.value = evt.posX || 0;
    posY.value = evt.posY || 0;
    nextTick(() => {
      show.value = true;
    });
  });
  onBeforeUnmount(() => {
    eventbus.off('contextmenu');
  });
});
</script>

<script lang="ts">
// Helpers and utilities for Contextmenu items

import router from '@/plugins/router';

import {
  ProviderFeature,
  MediaItem,
  QueueOption,
  MediaType,
  Playlist,
  Album,
  Track,
  ItemMapping,
  MediaItemType,
} from '@/plugins/api/interfaces';
import { $t } from '@/plugins/i18n';
import { itemIsAvailable } from '@/plugins/api/helpers';

export interface ContextMenuItem {
  label: string;
  labelArgs?: Array<string | number>;
  action?: CallableFunction;
  icon?: string;
  disabled?: boolean;
  hide?: boolean;
  selected?: boolean;
}

export const showContextMenuForMediaItem = async function (
  item: MediaItemType | MediaItemType[],
  parentItem?: MediaItem,
  posX = 0,
  posY = 0,
  includePlayItems = true,
) {
  // show ContextMenu for given MediaItem(s)
  const mediaItems: MediaItemType[] = Array.isArray(item) ? item : [item];
  if (mediaItems.length == 0) return;

  const menuItems: ContextMenuItem[] = [];

  if (includePlayItems) {
    menuItems.push(...getPlayMenuItems(mediaItems, parentItem));
  }

  // collect all contextMenuItems
  menuItems.push(...getContextMenuItems(mediaItems, parentItem));

  // open the contextmenu by emitting the event
  eventbus.emit('contextmenu', {
    items: menuItems,
    posX: posX,
    posY: posY,
  });
};

export const getPlayMenuItems = function (
  items: Array<MediaItemType | ItemMapping>,
  parentItem?: MediaItem,
) {
  const playMenuItems: ContextMenuItem[] = [];
  if (items.length == 0 || !itemIsAvailable(items[0])) {
    return playMenuItems;
  }
  if (!store.activePlayerId) return playMenuItems;
  if (items[0].media_type == MediaType.FOLDER) return playMenuItems;

  // Play from here...
  if (items.length == 1 && parentItem && parentItem.uri != items[0].uri) {
    // Play from here (playlist track)
    if (parentItem.media_type == MediaType.PLAYLIST) {
      playMenuItems.push({
        label: 'play_playlist_from',
        action: () => {
          api.playMedia(parentItem.uri, undefined, false, items[0].item_id);
        },
        icon: 'mdi-play-circle-outline',
        labelArgs: [],
        disabled: !store.activePlayerQueue,
      });
    }
    // Play from here (album track)
    if (parentItem.media_type == MediaType.ALBUM) {
      playMenuItems.push({
        label: 'play_album_from',
        action: () => {
          api.playMedia(parentItem.uri, undefined, false, items[0].item_id);
        },
        icon: 'mdi-play-circle-outline',
        labelArgs: [],
        disabled: !store.activePlayerQueue,
      });
    }
  }

  // Play NOW
  playMenuItems.push({
    label: 'play_now',
    action: () => {
      api.playMedia(
        items.map((x) => x.uri),
        QueueOption.PLAY,
      );
    },
    icon: 'mdi-play-circle-outline',
    labelArgs: [],
    disabled: !store.activePlayerQueue,
  });

  // replace now
  playMenuItems.push({
    label: 'play_replace',
    action: () => {
      api.playMedia(
        items.map((x) => x.uri),
        QueueOption.REPLACE,
      );
    },
    icon: 'mdi-play-circle-outline',
    labelArgs: [],
    disabled: !store.activePlayerQueue,
  });

  // Play NEXT
  if (items.length === 1 || items[0].media_type === MediaType.TRACK) {
    playMenuItems.push({
      label: 'play_next',
      action: () => {
        api.playMedia(
          items.map((x) => x.uri),
          QueueOption.NEXT,
        );
      },
      icon: 'mdi-skip-next-circle-outline',
      labelArgs: [],
      disabled: !store.activePlayerQueue,
    });
  }
  // Add to Queue
  playMenuItems.push({
    label: 'add_queue',
    action: () => {
      api.playMedia(
        items.map((x) => x.uri),
        QueueOption.ADD,
      );
    },
    icon: 'mdi-playlist-plus',
    labelArgs: [],
    disabled: !store.activePlayerQueue,
  });

  // Start Radio
  if (radioModeSupported(items[0])) {
    playMenuItems.push({
      label: 'play_radio',
      action: () => {
        api.playMedia(
          items.map((x) => x.uri),
          undefined,
          true,
        );
      },
      icon: 'mdi-radio-tower',
      labelArgs: [],
      disabled: !store.activePlayerQueue,
    });
  }

  return playMenuItems;
};

export const getContextMenuItems = function (
  items: Array<MediaItemType | ItemMapping>,
  parentItem?: MediaItem,
) {
  const contextMenuItems: ContextMenuItem[] = [];
  if (items.length == 0) {
    return contextMenuItems;
  }
  if (items[0].media_type == MediaType.FOLDER) return contextMenuItems;
  // show info
  if (
    items.length === 1 &&
    items[0] !== parentItem &&
    itemIsAvailable(items[0])
  ) {
    contextMenuItems.push({
      label: 'show_info',
      labelArgs: [],
      action: () => {
        router.push({
          name: items[0].media_type,
          params: {
            itemId: items[0].item_id,
            provider: items[0].provider,
          },
          query: {
            album: 'album' in items[0] ? (items[0].album as Album)?.uri : '',
          },
        });
      },
      icon: 'mdi-information-outline',
    });
  }

  // go to artist(s)
  if (
    items.length === 1 &&
    itemIsAvailable(items[0]) &&
    'artists' in items[0] &&
    (items[0] as Track | Album).artists.length === 1
  ) {
    for (const artist of (items[0] as Track).artists) {
      contextMenuItems.push({
        label: 'goto_artist',
        labelArgs: [artist.name],
        action: () => {
          router.push({
            name: 'artist',
            params: {
              itemId: artist.item_id,
              provider: artist.provider,
            },
          });
        },
        icon: 'mdi-account-music',
      });
    }
  }
  // go to album
  if (
    items.length === 1 &&
    itemIsAvailable(items[0]) &&
    'album' in items[0] &&
    (items[0] as Track).album
  ) {
    contextMenuItems.push({
      label: 'goto_album',
      labelArgs: [(items[0] as Track).album.name],
      action: () => {
        router.push({
          name: 'album',
          params: {
            itemId: (items[0] as Track).album.item_id,
            provider: (items[0] as Track).album.provider,
          },
        });
      },
      icon: 'mdi-album',
    });
  }
  // refresh item
  if (
    items.length === 1 &&
    (items[0] == parentItem || !itemIsAvailable(items[0]))
  ) {
    contextMenuItems.push({
      label: 'refresh_item',
      labelArgs: [],
      action: async () => {
        await api.refreshItem(items[0]);
        router.go(0);
      },
      icon: 'mdi-refresh',
    });
  }
  // add to library
  if (items[0].provider != 'library' && itemIsAvailable(items[0])) {
    contextMenuItems.push({
      label: 'add_library',
      labelArgs: [],
      action: () => {
        for (const item of items) api.addItemToLibrary(item);
      },
      icon: 'mdi-bookshelf',
    });
  }
  // remove from library
  if (items[0].provider == 'library') {
    contextMenuItems.push({
      label: 'remove_library',
      labelArgs: [],
      action: () => {
        if (!confirm($t('confirm_library_remove'))) return;
        for (const item of items)
          api.removeItemFromLibrary(item.media_type, item.item_id);
        if (items[0].item_id == parentItem?.item_id) router.go(-1);
        else router.go(0);
      },
      icon: 'mdi-bookshelf',
    });
  }
  // add to favorites
  if ('favorite' in items[0] && !items[0].favorite) {
    contextMenuItems.push({
      label: 'favorites_add',
      labelArgs: [],
      action: () => {
        for (const item of items) {
          api.addItemToFavorites(item);
        }
      },
      icon: 'mdi-heart-outline',
    });
  }
  // remove from favorites
  if ('favorite' in items[0] && items[0].favorite) {
    contextMenuItems.push({
      label: 'favorites_remove',
      labelArgs: [],
      action: () => {
        for (const item of items) {
          api.removeItemFromFavorites(item.media_type, item.item_id);
          (item as MediaItemType).favorite = false;
        }
      },
      icon: 'mdi-heart',
    });
  }
  // remove from playlist (playlist tracks only)
  if (parentItem && parentItem.media_type === MediaType.PLAYLIST) {
    const playlist = parentItem as Playlist;
    if (items[0].media_type === MediaType.TRACK && playlist.is_editable) {
      contextMenuItems.push({
        label: 'remove_playlist',
        labelArgs: [],
        action: () => {
          api.removePlaylistTracks(
            playlist.item_id,
            items.map((x) => (x as Track).position as number),
          );
        },
        icon: 'mdi-minus-circle-outline',
      });
    }
  }
  // add to playlist action (tracks only)
  if (
    (items[0].media_type === MediaType.TRACK ||
      items[0].media_type === MediaType.ALBUM) &&
    'provider_mappings' in items[0]
  ) {
    contextMenuItems.push({
      label: 'add_playlist',
      labelArgs: [],
      action: () => {
        eventbus.emit('playlistdialog', {
          items: items as MediaItemType[],
          parentItem: parentItem,
        });
      },
      icon: 'mdi-plus-circle-outline',
    });
  }
  return contextMenuItems;
};

const radioModeSupported = function (item: MediaItemType | ItemMapping) {
  if ('provider_mappings' in item) {
    for (const provId of item.provider_mappings) {
      if (
        api.providers[provId.provider_instance]?.supported_features.includes(
          ProviderFeature.SIMILAR_TRACKS,
        )
      )
        return true;
    }
  } else if (
    api.providers[item.provider]?.supported_features.includes(
      ProviderFeature.SIMILAR_TRACKS,
    )
  ) {
    return true;
  }
  // we also support a generic radio mode if we have ANY provider with similar track feature
  // and the track is (matched) in the library
  if (item.provider == 'library') {
    for (const prov of Object.values(api.providers)) {
      if (prov.supported_features.includes(ProviderFeature.SIMILAR_TRACKS))
        return true;
    }
  }

  return false;
};
</script>

<style scoped>
.menurow >>> .v-list-item__prepend {
  width: 45px;
  margin-left: -5px;
}

.menurow >>> .v-expansion-panel-title {
  padding: 0;
  padding-right: 10px;
  min-height: 40px !important;
  height: 40px !important;
}

.menurow >>> .v-expansion-panel-title--active {
  height: 40px !important;
}

.menurow >>> .v-expansion-panel-text__wrapper {
  padding: 0;
}
</style>
