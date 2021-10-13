<template>
    <div class="batch-edit-relate-box" v-bkloading="{ isLoading }">
        <jb-form
            ref="formRef"
            form-type="vertical"
            :model="formData"
            :rules="rules">
            <jb-form-item
                :label="$t('tag.范围')"
                required
                property="resourceTypeList">
                <bk-checkbox-group v-model="formData.resourceTypeList">
                    <bk-checkbox
                        class="mr10"
                        :value="5">
                        {{ $t('tag.作业') }}（{{ data.relatedTaskTemplateNum }}）
                    </bk-checkbox>
                    <bk-checkbox :value="1">
                        {{ $t('tag.脚本') }}（{{ data.relatedScriptNum }}）
                    </bk-checkbox>
                </bk-checkbox-group>
            </jb-form-item>
            <jb-form-item
                :label="$t('tag.标签')"
                style="margin-bottom: 0;">
                <div class="tag-panel">
                    <bk-input
                        class="tag-search"
                        :value="search"
                        :spellcheck="false"
                        left-icon="bk-icon icon-search"
                        @change="handleSearch" />
                    <div class="wrapper" style="height: 210px;">
                        <scroll-faker
                            v-if="renderList.length > 0"
                            ref="scrollRef">
                            <bk-checkbox-group
                                v-model="formData.operationList"
                                class="tag-list">
                                <bk-checkbox
                                    v-for="tagItem in renderList"
                                    :value="tagItem.id"
                                    class="tag-item"
                                    :key="tagItem.id">
                                    {{ tagItem.name }}
                                    <Icon
                                        v-if="tagItem.isNew"
                                        type="new"
                                        svg
                                        class="new-tag-flag" />
                                </bk-checkbox>
                            </bk-checkbox-group>
                        </scroll-faker>
                        <Empty
                            v-else-if="search"
                            type="search"
                            style="margin-top: 20px;">
                            <span>{{ $t('tag.搜索结果为空') }}，</span>
                            <bk-button
                                text
                                @click="handleClearSearch">
                                {{ $t('tag.清空搜索') }}
                            </bk-button>
                        </Empty>
                    </div>
                    <auth-component auth="tag/create">
                        <div class="tag-create" @click="handleNew">
                            <bk-icon
                                type="plus-circle"
                                style=" margin-right: 8px; font-size: 16px;" />
                            <span>{{ $t('tag.新增标签') }}</span>
                        </div>
                        <div slot="forbid" class="tag-create">
                            <bk-icon
                                type="plus-circle"
                                style=" margin-right: 8px; font-size: 16px;" />
                            <span>{{ $t('tag.新增标签') }}</span>
                        </div>
                    </auth-component>
                </div>
            </jb-form-item>
        </jb-form>
        <lower-component
            level="custom"
            :custom="isShowCreateTag">
            <operation-tag
                v-model="isShowCreateTag"
                @on-change="handleTagOperationChange" />
        </lower-component>
    </div>
</template>
<script>
    import {
        reactive,
        computed,
        ref,
        toRefs,
        onBeforeMount,
        getCurrentInstance,
    } from '@vue/composition-api';
    import _ from 'lodash';
    import I18n from '@/i18n';
    import TagManageService from '@service/tag-manage';
    import { encodeRegexp } from '@utils/assist';
    import OperationTag from '@components/operation-tag';

    export default {
        name: '',
        components: {
            OperationTag,
        },
        props: {
            data: {
                type: Object,
                required: true,
            },
        },
        setup (props, ctx) {
            const state = reactive({
                isLoading: true,
                isShowCreateTag: false,
                newTagList: [],
                wholeTagList: [],
                search: '',
                formData: {
                    resourceTypeList: [1, 5],
                    operationList: [],
                },
            });
            // 默认选中的 tag
            state.formData.operationList = [props.data.id];
            const formRef = ref(null);
            const scrollRef = ref(null);
            const isSubmitDisable = computed(() => props.data.relatedTaskTemplateNum + props.data.relatedScriptNum < 1);
            const { proxy } = getCurrentInstance();
            // 表单验证规则
            const rules = {
                resourceTypeList: [
                    {
                        validator: resourceTypeList => resourceTypeList.length > 0,
                        message: I18n.t('tag.范围不能为空'),
                        trigger: 'blur',
                    },
                ],
            };
            // 展示的 tag 列表数据
            const renderList = computed(() => {
                const allTagList = [...state.newTagList, ...state.wholeTagList];
                if (!state.search) {
                    return allTagList;
                }
                const searchReg = new RegExp(encodeRegexp(state.search), 'i');
                const result = allTagList.filter(item => searchReg.test(item.name));
                return Object.freeze(result);
            });
            /**
             * @desc 获取 tag 列表
             */
            const fetchData = () => {
                proxy.$request(TagManageService.fetchWholeList(), () => {
                    state.isLoading = true;
                })
                    .then((data) => {
                        // 排序
                        // 当前编辑的 tag 在最上面
                        state.wholeTagList = Object.freeze(data.reduce((result, item) => {
                            if (item.id === props.data.id) {
                                result.unshift(item);
                            } else {
                                result.push(item);
                            }
                            return result;
                        }, []));
                    })
                    .finally(() => {
                        state.isLoading = false;
                    });
            };
            /**
             * @desc 外部调用——刷新 tag 数据
             */
            const refresh = () => {
                this.fetchData();
            };
            /**
             * @desc 过滤 tag 列表
             */
            const handleSearch = _.debounce((search) => {
                state.search = _.trim(search);
            }, 300);
            /**
             * @desc 清除过滤项
             */
            const handleClearSearch = () => {
                state.search = '';
            };
            /**
             * @desc 新建 tag
             * @returns { Boolean }
             */
            const handleNew = () => {
                state.isShowCreateTag = true;
            };
            const handleTagOperationChange = (tag) => {
                tag.isNew = true;
                state.newTagList.unshift(tag);
                if (scrollRef.value) {
                    scrollRef.value.scrollTo(0, 0);
                }
            };
            /**
             * @desc 提交批量流转
             * @returns { Promise }
             */
            const submit = () => {
                if (isSubmitDisable.value) {
                    return Promise.resolve();
                }
                const {
                    operationList,
                    resourceTypeList,
                } = state.formData;

                const addTagIdList = operationList;
                const deleteTagIdList = [];
                if (!operationList.includes(props.data.id)) {
                    deleteTagIdList.push(props.data.id);
                }
                _.remove(addTagIdList, id => id === props.data.id);
                return formRef.value.validate()
                    .then(() => TagManageService.batchUpdate({
                        id: props.data.id,
                        resourceTypeList,
                        addTagIdList,
                        deleteTagIdList,
                    }))
                    .then(() => {
                        proxy.messageSuccess(I18n.t('tag.批量流转关联项成功'));
                        ctx.emit('on-change');
                    });
            };

            onBeforeMount(() => {
                fetchData();
            });

            return {
                ...toRefs(state),
                isSubmitDisable,
                formRef,
                scrollRef,
                rules,
                renderList,
                refresh,
                handleSearch,
                handleClearSearch,
                handleNew,
                handleTagOperationChange,
                submit,
            };
        },
    };
</script>
<style lang="postcss">
    .batch-edit-relate-box {
        padding-top: 5px;

        .tag-panel {
            display: flex;
            flex-direction: column;
            border: 1px solid #dcdee5;
            border-radius: 2px;
        }

        .tag-search {
            margin: 0 10px;

            &.bk-form-control {
                &.with-left-icon {
                    width: auto;

                    .left-icon {
                        left: 0;
                    }

                    .bk-form-input {
                        padding-left: 24px;
                    }
                }
            }

            .bk-form-input {
                border-color: transparent !important;
                border-bottom: 1px solid #c4c6cc !important;
            }
        }

        .wrapper {
            padding: 18px 0 18px 12px;

            .tag-list {
                display: flex;
                flex-direction: column;

                .tag-item {
                    margin-bottom: 16px;

                    &:last-child {
                        margin-bottom: 0;
                    }
                }
            }

            .new-tag-flag {
                margin-left: 5px;
                font-size: 18px;
                vertical-align: middle;
            }
        }

        .tag-create {
            height: 38px;
            padding-left: 10px;
            font-size: 12px;
            line-height: 38px;
            color: #63656e;
            cursor: pointer;
            background: #fafbfd;
            border-top: 1px solid #dcdee5;
        }
    }
</style>