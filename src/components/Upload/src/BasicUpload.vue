<template>
  <div>
    <a-button-group>
      <a-button
        v-if="!readonly"
        type="primary"
        @click="openUploadModal"
        preIcon="carbon:cloud-upload"
      >
        {{ t('component.upload.upload') }}
      </a-button>
      <Tooltip placement="bottom" v-if="showPreview">
        <template #title>
          {{ t('component.upload.uploaded') }}
          <template v-if="fileList.length">
            {{ fileList.length }}
          </template>
        </template>
        <a-button @click="openPreviewModal">
          <Icon icon="bi:eye" />
          <template v-if="fileList.length && showPreviewNumber">
            {{ fileList.length }}
          </template>
        </a-button>
      </Tooltip>
    </a-button-group>
    <UploadModal
      v-bind="bindValue"
      :previewFileList="fileList"
      @register="registerUploadModal"
      @change="handleChange"
      @delete="handleDelete"
    />
    <UploadPreviewModal
      :value="fileList"
      @register="registerPreviewModal"
      @change="handlePreviewChange"
      @delete="handleDelete"
    />
  </div>
</template>
<script lang="ts">
  import { defineComponent, ref, watch, unref, computed } from 'vue';
  import UploadModal from './UploadModal.vue';
  import UploadPreviewModal from './UploadPreviewModal.vue';
  import { Icon } from '/@/components/Icon';
  import { Tooltip } from 'ant-design-vue';
  import { useModal } from '/@/components/Modal';
  import { uploadContainerProps } from './props';
  import { omit } from 'lodash-es';
  import { useI18n } from '/@/hooks/web/useI18n';
  import { isArray } from '/@/utils/is';
  import { FileUpload, uploadFileList } from '/@/api/sys/upload';

  export default defineComponent({
    name: 'BasicUpload',
    components: { UploadModal, UploadPreviewModal, Icon, Tooltip },
    props: uploadContainerProps,
    emits: ['change', 'delete', 'update:value'],

    setup(props, { emit, attrs }) {
      const { t } = useI18n();
      const [registerUploadModal, { openModal: openUploadModal }] = useModal();
      const [registerPreviewModal, { openModal: openPreviewModal }] = useModal();

      const dataMap = ref<Object>({});
      const fileList = ref<FileUpload[]>([]);
      const fileListDel = ref<FileUpload[]>([]);

      const showPreview = computed(() => {
        const { emptyHidePreview } = props;
        if (!emptyHidePreview) return true;
        return emptyHidePreview ? fileList.value.length > 0 : true;
      });

      const bindValue = computed(() => {
        const value = { ...attrs, ...props };
        return omit(value, 'onChange');
      });

      watch(
        () => props.value,
        (value = '') => {
          dataMap.value = value;
        },
        { immediate: true },
      );

      watch(
        () => [props.bizKey, props.loadTime],
        () => {
          loadFileList();
        },
        { immediate: true },
      );

      function loadFileList() {
        fileList.value = [];
        if (props.bizKey != '') {
          uploadFileList({
            bizKey: props.bizKey,
            bizType: props.bizType,
          }).then((res) => {
            if (isArray(res)) {
              fileList.value = res;
            }
          });
        }
      }

      // 上传modal保存操作
      function handleChange(records: FileUpload[]) {
        fileList.value = [...unref(fileList), ...(records || [])];
        dataMap.value[props.bizType] = fileList.value.map((item) => item.id).join(',');
        emit('update:value', dataMap.value);
        emit('change', dataMap.value);
      }

      // 预览modal保存操作
      function handlePreviewChange(records: FileUpload[]) {
        fileList.value = [...(records || [])];
        dataMap.value[props.bizType] = fileList.value.map((item) => item.id).join(',');
        emit('update:value', dataMap.value);
        emit('change', dataMap.value);
      }

      function handleDelete(record: FileUpload) {
        fileListDel.value.push(record);
        dataMap.value[props.bizType + '__del'] = fileListDel.value.map((item) => item.id).join(',');
        emit('delete', record);
        emit('update:value', dataMap.value);
        emit('change', dataMap.value);
      }

      return {
        registerUploadModal,
        openUploadModal,
        handleChange,
        handlePreviewChange,
        registerPreviewModal,
        openPreviewModal,
        fileList,
        showPreview,
        bindValue,
        handleDelete,
        t,
      };
    },
  });
</script>
