<template>
  <div class="sing-toolbar">
    <character-menu-button>
      <div class="character-menu-toggle">
        <q-avatar
          v-if="selectedStyleIconPath"
          class="character-avatar"
          size="3.5rem"
        >
          <img :src="selectedStyleIconPath" class="character-avatar-icon" />
        </q-avatar>
        <div class="character-info">
          <div class="character-name">
            {{ selectedCharacterName }}
          </div>
          <div class="character-style">
            {{ selectedCharacterStyleDescription }}
          </div>
        </div>
        <q-icon
          name="arrow_drop_down"
          size="sm"
          class="character-menu-dropdown-icon"
        />
      </div>
    </character-menu-button>
    <div class="sing-player">
      <q-btn
        flat
        round
        class="sing-transport-button"
        icon="skip_previous"
        @click="goToZero"
      ></q-btn>
      <q-btn
        v-if="!nowPlaying"
        round
        class="sing-playback-button"
        icon="play_arrow"
        @click="play"
      ></q-btn>
      <q-btn
        v-else
        round
        class="sing-playback-button"
        icon="stop"
        @click="stop"
      ></q-btn>
      <div class="sing-playhead-position">{{ playheadPositionStr }}</div>
      <q-input
        type="number"
        :model-value="bpmInputBuffer"
        dense
        hide-bottom-space
        class="sing-tempo"
        @update:model-value="setBpmInputBuffer"
        @change="setTempo"
      >
        <template #prepend>
          <div></div>
        </template>
      </q-input>
      <q-input
        type="number"
        :model-value="beatsInputBuffer"
        dense
        hide-bottom-space
        class="sing-time-signature"
        @update:model-value="setBeatsInputBuffer"
        @change="setTimeSignature"
      >
        <template #prepend>
          <div></div>
        </template>
      </q-input>
      /
      <q-input
        type="number"
        :model-value="beatTypeInputBuffer"
        dense
        hide-bottom-space
        class="sing-time-signature"
        @update:model-value="setBeatTypeInputBuffer"
        @change="setTimeSignature"
      >
        <template #prepend>
          <div></div>
        </template>
      </q-input>
    </div>
    <div class="sing-setting">
      <q-slider v-model.number="volume" class="sing-volume" />
      <q-select
        v-model="snapTypeSelectModel"
        :options="snapTypeSelectOptions"
        color="primary"
        text-color="display-on-primary"
        outlined
        dense
        options-dense
        transition-show="none"
        transition-hide="none"
        class="sing-snap"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, watch, ref, onMounted, onUnmounted } from "vue";
import { useStore } from "@/store";
import {
  getSnapTypes,
  isTriplet,
  isValidBeatType,
  isValidBeats,
  isValidBpm,
} from "@/sing/domain";
import CharacterMenuButton from "@/components/Sing/CharacterMenuButton.vue";
import { getStyleDescription } from "@/sing/viewHelper";

const store = useStore();

const userOrderedCharacterInfos = computed(() =>
  store.getters.USER_ORDERED_CHARACTER_INFOS("singerLike")
);
const selectedCharacterInfo = computed(() => {
  const singer = store.state.tracks[0].singer;
  if (!userOrderedCharacterInfos.value || !singer) {
    return undefined;
  }
  return store.getters.CHARACTER_INFO(singer.engineId, singer.styleId);
});
const selectedCharacterName = computed(() => {
  return selectedCharacterInfo.value?.metas.speakerName;
});
const selectedCharacterStyleDescription = computed(() => {
  const style = selectedCharacterInfo.value?.metas.styles.find((style) => {
    const singer = store.state.tracks[0].singer;
    return (
      style.styleId === singer?.styleId && style.engineId === singer?.engineId
    );
  });
  return style != undefined ? getStyleDescription(style) : "";
});
const selectedStyleIconPath = computed(() => {
  const styles = selectedCharacterInfo.value?.metas.styles;
  const singer = store.state.tracks[0].singer;
  return styles?.find((style) => {
    return (
      style.styleId === singer?.styleId && style.engineId === singer?.engineId
    );
  })?.iconPath;
});

const bpmInputBuffer = ref(0);
const beatsInputBuffer = ref(0);
const beatTypeInputBuffer = ref(0);

const setBpmInputBuffer = (bpmStr: string | number | null) => {
  const bpm = Number(bpmStr);
  if (!isValidBpm(bpm)) {
    return;
  }
  bpmInputBuffer.value = bpm;
};
const setBeatsInputBuffer = (beatsStr: string | number | null) => {
  const beats = Number(beatsStr);
  if (!isValidBeats(beats)) {
    return;
  }
  beatsInputBuffer.value = beats;
};
const setBeatTypeInputBuffer = (beatTypeStr: string | number | null) => {
  const beatType = Number(beatTypeStr);
  if (!isValidBeatType(beatType)) {
    return;
  }
  beatTypeInputBuffer.value = beatType;
};

const playheadTicks = ref(0);

const playheadPositionStr = computed(() => {
  const ticks = playheadTicks.value;
  const time = store.getters.TICK_TO_SECOND(ticks);

  const intTime = Math.trunc(time);
  const min = Math.trunc(intTime / 60);
  const minStr = String(min).padStart(2, "0");
  const secStr = String(intTime - min * 60).padStart(2, "0");
  const milliSec = Math.trunc((time - intTime) * 1000);
  const milliSecStr = String(milliSec).padStart(3, "0");

  return `${minStr}:${secStr}.${milliSecStr}`;
});

const tempos = computed(() => store.state.tempos);
const timeSignatures = computed(() => store.state.timeSignatures);
const nowPlaying = computed(() => store.state.nowPlaying);

watch(
  tempos,
  () => {
    bpmInputBuffer.value = tempos.value[0].bpm;
  },
  { deep: true }
);
watch(
  timeSignatures,
  () => {
    beatsInputBuffer.value = timeSignatures.value[0].beats;
    beatTypeInputBuffer.value = timeSignatures.value[0].beatType;
  },
  { deep: true }
);

const setTempo = async () => {
  const bpm = bpmInputBuffer.value;
  if (bpm === 0) return;
  await store.dispatch("SET_TEMPO", {
    tempo: {
      position: 0,
      bpm,
    },
  });
};

const setTimeSignature = async () => {
  const beats = beatsInputBuffer.value;
  const beatType = beatTypeInputBuffer.value;
  if (beats === 0 || beatType === 0) return;
  await store.dispatch("SET_TIME_SIGNATURE", {
    timeSignature: {
      measureNumber: 1,
      beats,
      beatType,
    },
  });
};

const play = () => {
  store.dispatch("SING_PLAY_AUDIO");
};

const stop = () => {
  store.dispatch("SING_STOP_AUDIO");
};

const goToZero = () => {
  store.dispatch("SET_PLAYHEAD_POSITION", { position: 0 });
};

const volume = computed({
  get() {
    return store.state.volume * 100;
  },
  set(value: number) {
    store.dispatch("SET_VOLUME", { volume: value / 100 });
  },
});

const snapTypeSelectOptions = computed(() => {
  const tpqn = store.state.tpqn;
  return getSnapTypes(tpqn)
    .sort((a, b) => {
      if (isTriplet(a) === isTriplet(b)) {
        return a - b;
      } else {
        return isTriplet(a) ? 1 : -1;
      }
    })
    .map((snapType) => {
      if (isTriplet(snapType)) {
        return { snapType, label: `1/${(snapType / 3) * 2}（三連符）` };
      } else {
        return { snapType, label: `1/${snapType}` };
      }
    });
});
const snapTypeSelectModel = computed({
  get() {
    const snapType = store.state.sequencerSnapType;
    const selectOptions = snapTypeSelectOptions.value;
    return (
      selectOptions.find((value) => value.snapType === snapType) ??
      selectOptions[0]
    );
  },
  set(value) {
    store.dispatch("SET_SNAP_TYPE", {
      snapType: value.snapType,
    });
  },
});

const playheadPositionChangeListener = (position: number) => {
  playheadTicks.value = position;
};

onMounted(() => {
  store.dispatch("ADD_PLAYHEAD_POSITION_CHANGE_LISTENER", {
    listener: playheadPositionChangeListener,
  });
});

onUnmounted(() => {
  store.dispatch("REMOVE_PLAYHEAD_POSITION_CHANGE_LISTENER", {
    listener: playheadPositionChangeListener,
  });
});
</script>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.character-menu-toggle {
  align-items: center;
  display: flex;
  padding: 0.25rem 0.5rem 0.25rem 0.25rem;
  position: relative;
}
.character-avatar-icon {
  display: block;
  height: 100%;
  object-fit: cover;
  width: 100%;
}

.character-info {
  align-items: start;
  display: flex;
  flex-direction: column;
  margin-left: 0.5rem;
  text-align: left;
  justify-content: center;
  white-space: nowrap;
}
.character-name {
  font-size: 0.875rem;
  font-weight: bold;
  line-height: 1rem;
  padding-top: 0.5rem;
}

.character-style {
  color: rgba(colors.$display-rgb, 0.6);
  font-size: 0.75rem;
  font-weight: bold;
  line-height: 1rem;
}

.character-menu-dropdown-icon {
  color: rgba(colors.$display-rgb, 0.8);
  margin-left: 0.25rem;
}
.sing-toolbar {
  background: colors.$sing-toolbar;
  align-items: center;
  display: flex;
  justify-content: space-between;
  min-height: 56px;
  padding: 0 16px 0 0;
  width: 100%;
}

.sing-player {
  align-items: center;
  display: flex;
}

.sing-transport-button {
  margin: 0 1px;
}

.sing-playback-button {
  margin: 0 4px;
}

.sing-tempo {
  margin-left: 16px;
  margin-right: 4px;
  width: 64px;
}

.sing-time-signature {
  margin: 0 4px;
  width: 36px;
}

.sing-playhead-position {
  font-size: 18px;
  margin-left: 14px;
  margin-right: 4px;
  min-width: 82px;
}

.sing-setting {
  align-items: center;
  display: flex;
}

.sing-volume {
  margin-right: 16px;
  width: 72px;
}

.sing-snap {
  margin-right: 2px;
  min-width: 160px;
}
</style>
