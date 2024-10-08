<template>
  <a-modal
    v-model:visible="showNotify"
    :title="$t('common.operation.notify')"
    :on-before-ok="handleNotify"
    :on-before-close="
      notifyFormRef?.resetFields(['time', 'place', 'rest', 'meeting_id'])
    "
    :width="width < 666 ? '90%' : '600px'"
  >
    <a-form
      ref="notifyFormRef"
      :model="formData"
      layout="vertical"
      :rules="preview.rules"
    >
      <a-form-item class="max-sm:mb-3">
        <template #label>
          <span>{{ $t('candidate.receiver') }}</span>
          <span
            v-show="props.type === 'Reject'"
            class="text-[rgb(var(--danger-6))]"
            >{{ `(${$t('common.status.rejected')})` }}</span
          >
        </template>
        <a-input-tag
          :model-value="candidates.map(({ name }) => name)"
          readonly
        />
      </a-form-item>

      <div class="flex sm:gap-2 justify-between w-full max-sm:flex-col">
        <a-form-item
          class="max-sm:mb-3"
          field="next"
          :label="$t('common.user.nextStage')"
          asterisk-position="end"
          validate-trigger="change"
        >
          <a-select v-model:model-value="formData.next">
            <a-option
              v-for="item in nextValidSteps"
              :key="item"
              :value="item"
              :title="$t(`common.steps.${item}`)"
              >{{ $t(`common.steps.${item}`) }}</a-option
            >
          </a-select>
        </a-form-item>

        <a-form-item
          class="max-sm:mb-3"
          field="time"
          :disabled="!preview.notDisable.includes('time')"
          :label="$t('common.time')"
          asterisk-position="end"
          validate-trigger="change"
        >
          <a-date-picker
            v-model="formData.time"
            show-time
            format="YYYY-MM-DD HH:mm"
            value-format="YYYY-MM-DD HH:mm:00"
          />
        </a-form-item>

        <a-form-item
          class="max-sm:mb-3"
          field="meeting_id"
          :disabled="!preview.notDisable.includes('meeting_id')"
          :label="$t('common.sms.meetingId')"
          asterisk-position="end"
          validate-trigger="change"
        >
          <a-input v-model="formData.meeting_id" />
        </a-form-item>
      </div>
      <div class="flex gap-2 justify-between w-full flex-col sm:flex-row">
        <a-form-item
          class="max-sm:mb-3"
          field="place"
          :disabled="!preview.notDisable.includes('place')"
          :label="$t('common.sms.place')"
          asterisk-position="end"
          validate-trigger="change"
        >
          <a-input v-model="formData.place" />
        </a-form-item>
        <a-form-item
          class="max-sm:mb-3"
          field="rest"
          :disabled="!preview.notDisable.includes('rest')"
          :label="$t('common.sms.rest')"
          asterisk-position="end"
          validate-trigger="change"
        >
          <a-input v-model="formData.rest" />
        </a-form-item>
      </div>
      <a-form-item class="max-sm:mb-3" :label="$t('common.sms.example')">
        <a-scrollbar
          class="flex flex-col w-full max-h-24 overflow-y-auto"
          outer-class="w-full"
        >
          <a-scrollbar
            v-for="candidate in props.candidates"
            :key="candidate.aid"
            class="rounded-md border-2 px-4 py-3 break-all overflow-y-auto max-h-20"
            outer-class="w-full pb-4"
          >
            <i18n-t :keypath="preview.i18nKey || ''" tag="div">
              <template #name>
                <span class="text-[rgb(var(--primary-6))]">{{
                  candidate.name || '{ }'
                }}</span>
              </template>
              <template #recruitment_name>
                <span class="text-[rgb(var(--primary-6))]">{{
                  $t(recNameI18nKey, {
                    defaultValue: recNameI18nKey,
                  })
                }}</span>
              </template>
              <template #group>
                <span class="text-[rgb(var(--primary-6))]">{{ group }}</span>
              </template>
              <template #current>
                <span class="text-[rgb(var(--primary-6))]">{{
                  curStep < recruitSteps.length &&
                  $t(recruitSteps[props.curStep].i18Key)
                }}</span>
              </template>
              <template #next>
                <span class="text-[rgb(var(--primary-6))]">{{
                  formData.next && $t(`common.steps.${formData.next}`)
                }}</span>
              </template>
              <template #meeting_id>
                <span class="text-[rgb(var(--primary-6))]">{{
                  formData.meeting_id || '{ }'
                }}</span>
              </template>
              <template #time>
                <span class="text-[rgb(var(--primary-6))]">{{
                  formData.time || '{ }'
                }}</span>
              </template>
              <template #place>
                <span class="text-[rgb(var(--primary-6))]">{{
                  formData.place || '{ }'
                }}</span>
              </template>
              <template #rest>
                <span
                  v-if="formData.rest"
                  class="text-[rgb(var(--primary-6))]"
                  >{{ formData.rest }}</span
                >
                <i18n-t
                  v-else-if="props.type === 'Accept'"
                  :keypath="preview.restI18nKey ?? ''"
                  tag="span"
                >
                  <template #next>
                    <span class="text-[rgb(var(--primary-6))]">{{
                      formData.next && $t(`common.steps.${formData.next}`)
                    }}</span>
                  </template>
                  <template #time>
                    <span class="text-[rgb(var(--primary-6))]">{{
                      formData.time || '{ }'
                    }}</span>
                  </template>
                  <template #place>
                    <span class="text-[rgb(var(--primary-6))]">{{
                      formData.place || '{ }'
                    }}</span>
                  </template>
                  <template #group>
                    <span class="text-[rgb(var(--primary-6))]">{{
                      group
                    }}</span>
                  </template>
                </i18n-t>
              </template>
              <!-- 组面、群面无需传时间，其时间已在面试管理分配，example_time仅用于展示占位 -->
              <template #example_time>
                <!-- 3-(在线)组面 6-(在线)群面 -->
                <span
                  v-if="
                    recruitSteps[3].value.includes(formData.next) &&
                    candidate.groupInterviewTime
                  "
                  class="text-[rgb(var(--primary-6))]"
                >
                  {{
                    dayjs(candidate.groupInterviewTime).format(
                      'YYYY-MM-DD HH:mm:00',
                    )
                  }}
                </span>
                <span
                  v-else-if="
                    recruitSteps[6].value.includes(formData.next) &&
                    candidate.teamInterviewTime
                  "
                  class="text-[rgb(var(--primary-6))]"
                >
                  {{
                    dayjs(candidate.teamInterviewTime).format(
                      'YYYY-MM-DD HH:mm:00',
                    )
                  }}
                </span>
                <span
                  v-else
                  class="cursor-pointer text-[rgb(var(--warning-4))] hover:text-[rgb(var(--warning-5))]"
                  @click="
                    $router.push('/interview/management');
                    showNotify = false;
                  "
                  >{{ `\{${$t('common.status.waitForDistribution')}\}` }}</span
                >
              </template>
              <template #online_interview_type>
                {{ $t(`common.steps.${formData.next}`) }}
              </template>
            </i18n-t>
          </a-scrollbar>
        </a-scrollbar>
      </a-form-item>
    </a-form>
  </a-modal>
</template>

<script setup lang="ts">
import { ref, PropType, watch, computed } from 'vue';
import { Group, recruitSteps, Step, SMSTemplate } from '@/constants/team';
import { sendSms } from '@/api';
import { groupBy } from 'lodash';
import { Message } from '@arco-design/web-vue';
import { useI18n } from 'vue-i18n';
import dayjs from 'dayjs';
import { getRecruitmentName } from '@/utils';
import useWindowResize from '@/hooks/resize';
import useRecruitmentStore from '@/store/modules/recruitment';

const { t } = useI18n();
const { width } = useWindowResize();

const props = defineProps({
  candidates: {
    type: Array as PropType<
      {
        name: string;
        aid: string;
        step: Step;
        groupInterviewTime: string;
        teamInterviewTime: string;
      }[]
    >,
    required: true,
  },
  curStep: {
    type: Number,
    required: true,
  },
  type: {
    type: String as PropType<'Accept' | 'Reject'>,
    required: true,
  },
  group: {
    type: String as PropType<Group>,
    required: true,
  },
});

const recStore = useRecruitmentStore();

const recName = computed(() => recStore.currentRec?.name ?? '');

const recNameI18nKey = computed(() => getRecruitmentName(t, recName.value));

const showNotify = defineModel<boolean>('showNotify', {
  type: Boolean,
  default: false,
  required: true,
});

const nextValidSteps = computed(() => {
  const arr: Step[] = [];
  recruitSteps
    .slice(props.curStep + 1)
    .forEach(({ value }) => arr.push(...value));
  return arr;
});
const formData = ref({
  next: nextValidSteps.value[0], // 下一步流程
  time: '', // 笔试/面试/熬测时间
  place: '', // 地点
  meeting_id: '', // 在线面试的会议id
  rest: '', // 补充信息
});
watch(nextValidSteps, () => {
  [formData.value.next] = nextValidSteps.value;
});

const preview = computed(() => {
  if (props.type === 'Reject')
    return {
      ...SMSTemplate[0],
      rules: {},
      notDisable: SMSTemplate[0].required.concat(['rest']),
    };
  const template = SMSTemplate.find(({ match }) =>
    match.includes(formData.value.next || Step.Pass),
  );
  return {
    ...template,
    rules: template
      ? Object.fromEntries(
          template.required.map((key) => [
            key,
            [{ required: !formData.value.rest || !template.restI18nKey }],
          ]),
        )
      : {},
    notDisable:
      template?.required.concat(template.restI18nKey ? ['rest'] : []) ?? [],
  };
});

const notifyFormRef = ref<any>(null);

watch(
  () => formData.value.next,
  () => {
    notifyFormRef.value?.clearValidate();
  },
);

const handleNotify = async () => {
  const validateError = await notifyFormRef.value?.validate();
  if (validateError) return false;
  if (
    recruitSteps[3].value.includes(formData.value.next) &&
    !props.candidates.every(({ groupInterviewTime }) => groupInterviewTime)
  ) {
    Message.error(t('candidate.requireAllocateTime'));
    return false;
  }
  if (
    recruitSteps[6].value.includes(formData.value.next) &&
    !props.candidates.every(({ teamInterviewTime }) => teamInterviewTime)
  ) {
    Message.error(t('candidate.requireAllocateTime'));
    return false;
  }

  const res = groupBy(props.candidates, ({ step }) => step);
  const reqs = Object.entries(res).map(([current, arr]) => {
    const aids = arr.map(({ aid }) => aid);
    return sendSms({
      type: props.type,
      current: current as Step,
      ...formData.value,
      aids,
    });
  });
  const resp = await Promise.all(reqs);
  if (!resp.every((x) => x)) return false;
  Message.success(t('common.result.sendSuccess'));
  notifyFormRef.value?.resetFields();
  [formData.value.next] = nextValidSteps.value;
  return true;
};
</script>

<style scoped lang="less"></style>
