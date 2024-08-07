<template>
  <div class="flex justify-end gap-2 items-center">
    <a-button
      type="primary"
      :disabled="props.curStep >= recruitSteps.length - 1 || !candidates.length"
      :size="buttonSize"
      class="max-sm:rounded-full rounded-none"
      @click="openSwitchStage"
    >
      {{ $t('common.operation.switchStage') }}
    </a-button>
    <a-button
      status="danger"
      :disabled="props.curStep >= recruitSteps.length || !candidates.length"
      :size="buttonSize"
      class="max-sm:rounded-full rounded-none"
      @click="openTerminate"
    >
      {{ $t('common.operation.terminate') }}
    </a-button>
    <a-button
      type="outline"
      class="max-sm:hidden"
      :size="buttonSize"
      :disabled="props.curStep >= recruitSteps.length || !candidates.length"
      @click="openNotify"
    >
      <template #icon> <icon-plus /> </template>
      {{ $t('common.operation.sendNotification') }}
    </a-button>
    <a-button
      type="outline"
      class="sm:hidden rounded-full"
      :size="buttonSize"
      :disabled="!candidates.length"
      @click="openNotify"
    >
      {{ $t('common.operation.sendNotification') }}
    </a-button>
  </div>
  <a-modal
    v-model:visible="showSwitchStage"
    simple
    message-type="info"
    :title="$t('common.operation.switchStage')"
    :on-before-ok="handleSwitchStage"
  >
    <div>
      <a-alert type="warning" class="mb-3">{{
        $t('candidate.warnNotify')
      }}</a-alert>
      <a-form class="w-full" :model="{}" layout="vertical">
        <a-form-item :label="$t('common.candidate')">
          <a-input-tag v-model="candidateNames" :max-tag-count="4" readonly />
        </a-form-item>
        <a-form-item field="data">
          <template #label>
            <i18n-t keypath="candidate.switchStage" tag="div">
              <template #cur>
                <span class="text-[rgb(var(--primary-6))]">{{
                  curStep < recruitSteps.length &&
                  $t(recruitSteps[curStep].i18Key)
                }}</span>
              </template>
            </i18n-t>
          </template>
          <a-select
            v-model:model-value="nextStep"
            class="text-[rgb(var(--primary-6))]"
          >
            <a-option
              v-for="item in Object.values(Step)"
              :key="item"
              :value="item"
              :disabled="recruitSteps[props.curStep]?.value.includes(item)"
              :title="$t(`common.steps.${item}`)"
              >{{ $t(`common.steps.${item}`) }}</a-option
            >
          </a-select>
        </a-form-item>
      </a-form>
    </div>
  </a-modal>
  <a-modal
    v-model:visible="showTerminate"
    simple
    message-type="warning"
    :title="$t('common.operation.confirmTerminate')"
    :on-before-ok="handleTerminate"
  >
    <div class="text-center">
      <i18n-t keypath="candidate.terminate" tag="div">
        <template #num>
          <span class="text-[rgb(var(--primary-6))]">{{
            candidates.length
          }}</span>
        </template>
      </i18n-t>
    </div>
  </a-modal>
  <notification-modal
    v-if="curStep <= recruitSteps.length"
    v-model:showNotify="showNotify"
    :candidates="props.candidates"
    :cur-step="props.curStep"
    :group="props.group"
    :type="allAccepted ? 'Accept' : 'Reject'"
  ></notification-modal>
</template>

<script setup lang="ts">
import { ref, PropType, computed, watch } from 'vue';
import { recruitSteps, Step, Group } from '@/constants/team';
import NotificationModal from '@/views/components/notification-modal.vue';
import { updateApplicationStep, rejectApplication } from '@/api';
import useRecruitmentStore from '@/store/modules/recruitment';
import { Message } from '@arco-design/web-vue';
import { useI18n } from 'vue-i18n';
import useWindowResize from '@/hooks/resize';

const { widthType } = useWindowResize();
const buttonSize = computed(() =>
  widthType.value === 'sm' ? 'mini' : 'medium',
);

const { t } = useI18n();

const recStore = useRecruitmentStore();

const props = defineProps({
  candidates: {
    type: Array as PropType<
      {
        name: string;
        aid: string;
        step: Step;
        abandoned: boolean;
        rejected: boolean;
        groupInterviewTime: string;
        teamInterviewTime: string;
      }[]
    >,
    default: () => [],
    required: true,
  },
  curStep: {
    type: Number,
    required: true,
  },
  group: {
    type: String as PropType<Group>,
    required: true,
  },
  onDone: {
    type: Function,
  },
});

const candidateNames = computed(() => props.candidates.map(({ name }) => name));

const allAccepted = computed(() =>
  props.candidates.every(({ abandoned, rejected }) => !abandoned && !rejected),
);
const allRejected = computed(() =>
  props.candidates.every(({ rejected }) => rejected),
);

const showSwitchStage = ref(false);
const showTerminate = ref(false);
const showNotify = ref(false);

const nextStep = ref(recruitSteps[props.curStep + 1]?.value[0] ?? Step.Pass);
watch(
  () => props.curStep,
  () => {
    [nextStep.value] = recruitSteps[props.curStep + 1]?.value ?? [Step.Pass];
  },
);

const openSwitchStage = () => {
  if (!allAccepted.value) {
    Message.error(t('candidate.noAbandonedRejected'));
    return;
  }
  showSwitchStage.value = true;
};

const handleSwitchStage = async () => {
  let res;
  try {
    res = await Promise.all(
      props.candidates.map(({ aid, step }) =>
        updateApplicationStep(aid, {
          from: step,
          to: nextStep.value,
        }),
      ),
    );
    return true;
  } catch (err) {
    return false;
  } finally {
    if (res?.every((x) => x))
      Message.success(t('common.result.switchStageSuccess'));
    props.onDone?.();
    recStore.refresh();
  }
};

const openTerminate = () => {
  if (!allAccepted.value) {
    Message.error(t('candidate.noAbandonedRejected'));
    return;
  }
  showTerminate.value = true;
};

const handleTerminate = async () => {
  let res;
  try {
    res = await Promise.all(
      props.candidates.map(({ aid }) => rejectApplication(aid)),
    );
    return true;
  } catch (err) {
    return false;
  } finally {
    if (res?.every((x) => x))
      Message.success(t('common.result.terminateSuccess'));
    props.onDone?.();
    recStore.refresh();
  }
};

const openNotify = () => {
  if (props.candidates.some(({ abandoned }) => abandoned)) {
    Message.error(t('candidate.noAbandoned'));
    return;
  }
  if (!allAccepted.value && !allRejected.value) {
    Message.error(t('candidate.noPassRejected'));
    return;
  }
  showNotify.value = true;
};
</script>

<style scoped lang="less"></style>
