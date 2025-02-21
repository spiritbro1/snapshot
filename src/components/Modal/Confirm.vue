<script setup>
import { computed, inject, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { shorten, getChoiceString, explorerUrl } from '@/helpers/utils';
import { useClient } from '@/composables/useClient';
import { useIntl } from '@/composables/useIntl';
import { getPower } from '../../helpers/snapshot';
import { useWeb3 } from '../../composables/useWeb3';

const { web3Account } = useWeb3();

const vp = ref(0);
const vpByStrategy = ref([]);
const vpLoading = ref(false);
const vpLoaded = ref(false);

const props = defineProps({
  open: Boolean,
  space: Object,
  proposal: Object,
  selectedChoices: [Object, Number],
  snapshot: String,
  strategies: Object
});

const emit = defineEmits(['reload', 'close']);

const { t } = useI18n();
const notify = inject('notify');
const { send, clientLoading } = useClient();
const format = getChoiceString;
const { formatNumber, formatCompactNumber } = useIntl();

const symbols = computed(() =>
  props.strategies.map(strategy => strategy.params.symbol)
);

async function handleSubmit() {
  const result = await send(props.space, 'vote', {
    proposal: props.proposal,
    choice: props.selectedChoices,
    metadata: {}
  });
  console.log('Result', result);
  if (result.id) {
    notify(['green', t('notify.voteSuccessful')]);
    emit('reload');
    emit('close');
  }
}

watch(
  () => [props.open, web3Account.value],
  async () => {
    if (props.open === false) return;
    if (vpLoaded.value) return;
    vpLoading.value = true;
    const response = await getPower(
      props.space,
      web3Account.value,
      props.proposal
    );
    vp.value = response.totalScore;
    vpByStrategy.value = response.scores;
    vpLoading.value = false;
    vpLoaded.value = true;
  }
);
</script>

<template>
  <UiModal :open="open" @close="$emit('close')" class="flex">
    <template v-slot:header>
      <h3>{{ $t('confirmVote') }}</h3>
    </template>
    <div class="flex flex-col flex-auto">
      <h4 class="m-4 mb-0 text-center">
        {{
          $tc('sureToVote', [
            shorten(format(proposal, selectedChoices), 'choice')
          ])
        }}
        <br />
        {{ $t('cannotBeUndone') }}
      </h4>
      <div class="m-4 p-4 border rounded-md link-color">
        <div class="flex">
          <span v-text="$t('options')" class="flex-auto text-color mr-1" />
          <span
            v-tippy="{
              content:
                format(proposal, selectedChoices).length > 30
                  ? format(proposal, selectedChoices)
                  : null
            }"
            class="text-right ml-4 truncated"
          >
            {{ format(proposal, selectedChoices) }}
          </span>
        </div>
        <div class="flex">
          <span v-text="$t('snapshot')" class="flex-auto text-color mr-1" />
          <a
            :href="explorerUrl(proposal.network, proposal.snapshot, 'block')"
            target="_blank"
            class="float-right"
          >
            {{ formatNumber(proposal.snapshot) }}
            <Icon name="external-link" class="ml-1" />
          </a>
        </div>
        <div class="flex">
          <span v-text="$t('votingPower')" class="flex-auto text-color mr-1" />
          <span
            v-if="vpLoaded && !vpLoading"
            v-tippy="{
              content: vpByStrategy
                .map(
                  (score, index) =>
                    `${formatCompactNumber(score)} ${symbols[index]}`
                )
                .join(' + ')
            }"
          >
            {{ formatCompactNumber(vp) }}
            {{ shorten(space.symbol, 'symbol') }}
          </span>
          <span v-else><UiLoading /></span>
          <a
            v-if="vp === 0 && vpLoaded && !vpLoading"
            target="_blank"
            href="https://github.com/snapshot-labs/snapshot/discussions/1015#discussioncomment-1599447"
            class="inline-block ml-1"
          >
            <Icon name="info" size="24" class="text-color" />
          </a>
        </div>
      </div>
    </div>
    <template v-slot:footer>
      <div class="w-2/4 float-left pr-2">
        <UiButton @click="$emit('close')" type="button" class="w-full">
          {{ $t('cancel') }}
        </UiButton>
      </div>
      <div class="w-2/4 float-left pl-2">
        <UiButton
          :disabled="vp === 0 || clientLoading"
          :loading="clientLoading"
          @click="handleSubmit"
          type="submit"
          class="w-full"
          primary
        >
          {{ $t('proposal.vote') }}
        </UiButton>
      </div>
    </template>
  </UiModal>
</template>
