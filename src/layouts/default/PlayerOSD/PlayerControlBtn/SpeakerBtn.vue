<template>
  <!-- active player -->
  <Button
    variant="icon"
    :ripple="false"
    :icon="responsive && !getBreakpointValue('bp6') ? true : false"
    @click="store.showPlayersMenu = true"
  >
    <v-badge
      v-if="store.activePlayer?.group_childs.length"
      size="small"
      :content="curGroupPlayers"
    >
      <v-icon :color="props.color ? color : ''" :size="24"
        >mdi-speaker-multiple</v-icon
      >
    </v-badge>
    <v-icon v-else :color="props.color ? color : ''" :size="24"
      >mdi-speaker</v-icon
    >
    <span
      v-if="!responsive || getBreakpointValue('bp6')"
      class="line-clamp-1 no_transform"
    >
      {{
        truncateString(
          store.activePlayerQueue?.display_name || $t('no_player'),
          getBreakpointValue('bp7') ? 16 : 8,
        )
      }}
    </span>
  </Button>
</template>

<script setup lang="ts">
import { computed } from 'vue';

import api from '@/plugins/api';
import { store } from '@/plugins/store';
import { getBreakpointValue } from '@/plugins/breakpoint';
import Button from '@/components/mods/Button.vue';
import { isColorDark, truncateString } from '@/helpers/utils';

// properties
export interface Props {
  color?: string;
  responsive?: boolean;
}
const props = defineProps<Props>();

// computed properties
const curGroupPlayers = computed(() => {
  if (!store.activePlayer) {
    return 0;
  }
  let count = 0;
  for (const groupChildId of store.activePlayer.group_childs) {
    const volumeChild = api?.players[groupChildId];
    if (volumeChild && volumeChild.available && volumeChild.powered) {
      count++;
    }
  }
  return count;
});
</script>

<style>
.no_transform {
  text-transform: none;
}
</style>
