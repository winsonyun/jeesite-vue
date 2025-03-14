<!--
 * Copyright (c) 2013-Now http://jeesite.com All rights reserved.
 * No deletion without permission, or be held responsible to law.
 * @description 支持字典类型、支持下拉框标签返回、支持 API 接口
 * @author Vben、ThinkGem
-->
<template>
  <div class="jeesite-tree-select">
    <TreeSelect
      v-bind="getAttrs"
      v-model:value="state"
      :treeData="treeDataRef"
      @click="handleFetch"
    >
      <template #[item]="data" v-for="item in Object.keys($slots)">
        <slot :name="item" v-bind="data || {}"></slot>
      </template>
      <template #notFoundContent v-if="loading">
        <span>
          <LoadingOutlined spin class="mr-1" />
          {{ t('component.form.apiSelectNotFound') }}
        </span>
      </template>
    </TreeSelect>
  </div>
</template>
<script lang="ts">
  import { defineComponent, ref, unref, computed, watch, onMounted } from 'vue';
  import { TreeSelect } from 'ant-design-vue';
  import { isEmpty, isFunction } from '/@/utils/is';
  import { propTypes } from '/@/utils/propTypes';
  import { listToTree } from '/@/utils/helper/treeHelper';
  import { useAttrs } from '/@/hooks/core/useAttrs';
  import { useRuleFormItem } from '/@/hooks/component/useFormItem';
  import { get } from 'lodash-es';
  import { LoadingOutlined } from '@ant-design/icons-vue';
  import { useI18n } from '/@/hooks/web/useI18n';
  import { useDict } from '/@/components/Dict';
  import { TreeItem } from '/@/components/Tree';

  export default defineComponent({
    name: 'JeeSiteTreeSelect',
    components: { TreeSelect, LoadingOutlined },
    // inheritAttrs: false,
    props: {
      value: {
        type: [Array, Object, String, Number] as PropType<Array<any> | object | string | number>,
      },
      labelValue: {
        type: [Array, Object, String, Number] as PropType<Array<any> | object | string | number>,
      },
      labelInValue: propTypes.bool,
      treeData: {
        type: Array as PropType<Recordable[] | TreeItem[]>,
        default: () => [],
      },
      api: {
        type: Function as PropType<(arg?: Recordable) => Promise<Recordable[] | TreeItem[]>>,
        default: null,
      },
      params: {
        type: Object as PropType<Recordable>,
        default: () => ({}),
      },
      isDisable: {
        type: Function as PropType<(node: Recordable) => boolean>,
        default: null,
      },
      resultField: propTypes.string.def(''),
      immediate: propTypes.bool.def(false),
      dictType: propTypes.string,
      treeCheckable: propTypes.bool,
      treeDataSimpleMode: propTypes.bool.def(true),
      canSelectParent: propTypes.bool.def(true),
    },
    emits: ['options-change', 'change', 'click'],
    setup(props, { emit }) {
      const { t } = useI18n();
      const attrs = useAttrs();
      const treeDataRef = ref<Recordable[]>(props.treeData);
      const isFirstLoad = ref<Boolean>(false);
      const loading = ref<Boolean>(false);

      const getAttrs = computed(() => {
        return {
          showSearch: true,
          treeNodeFilterProp: 'title',
          replaceFields: {
            value: props.dictType ? 'value' : 'id',
            title: 'name',
          },
          dropdownStyle: { maxHeight: '300px' },
          getPopupContainer: () => document.body,
          ...unref(attrs),
          ...(props as Recordable),
          treeDataSimpleMode: false,
        };
      });

      const [state] = useRuleFormItem(props);

      if (!isEmpty(props.dictType)) {
        const { initSelectTreeData } = useDict();
        initSelectTreeData(treeDataRef, props.dictType, true);
      }

      watch(
        () => props.treeData,
        () => {
          treeDataRef.value = getTreeData(props.treeData);
          emit('options-change', unref(treeDataRef));
        },
      );

      watch(
        () => props.params,
        () => {
          isFirstLoad.value && fetch();
        },
        { deep: true },
      );

      watch(
        () => props.immediate,
        (v) => {
          v && !isFirstLoad.value && fetch();
        },
      );

      onMounted(async () => {
        if (props.treeData && props.treeData.length > 0) {
          treeDataRef.value = getTreeData(props.treeData);
        }
        if (props.immediate) {
          await fetch();
          isFirstLoad.value = true;
        }
      });

      async function fetch() {
        const { api } = props;
        if (!api || !isFunction(api)) return;
        treeDataRef.value = [];
        try {
          loading.value = true;
          let res = await api(props.params);
          if (props.resultField) {
            res = get(res, props.resultField) || [];
          }
          if (Array.isArray(res)) {
            treeDataRef.value = getTreeData(res);
          }
          emit('options-change', unref(treeDataRef));
        } catch (error) {
          console.warn(error);
        } finally {
          loading.value = false;
        }
      }

      function getTreeData(treeData: Recordable[]) {
        if (props.treeDataSimpleMode) {
          return listToTree(treeData, {
            callback: (parent, node) => {
              if (props.isDisable && node) {
                node.disabled = props.isDisable(node);
              }
              if (!props.canSelectParent && parent) {
                if (parent.children && parent.children.length > 0) {
                  parent.disabled = true;
                }
              }
            },
          });
        }
        return treeData;
      }

      async function handleFetch() {
        if (!props.immediate && !unref(isFirstLoad)) {
          await fetch();
          isFirstLoad.value = true;
        }
        emit('click');
      }

      return { t, getAttrs, state, treeDataRef, loading, handleFetch };
    },
  });
</script>
<style lang="less">
  @prefix-cls: ~'jeesite-tree-select';

  .@{prefix-cls} {
    width: 100%;

    .ant-select-tree {
      padding: 4px;

      li {
        margin: 3px 0;

        .ant-select-tree-switcher {
          line-height: normal;
          vertical-align: baseline;
        }
      }
    }
  }
</style>
