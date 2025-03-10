<template>
	<div>
		<v-fancy-select v-model="interfaceId" class="select" :items="selectItems" />

		<v-notice v-if="interfaceId && !selectedInterface" class="not-found" type="danger">
			{{ t('interface_not_found', { interface: interfaceId }) }}
			<div class="spacer" />
			<button @click="interfaceId = null">{{ t('reset_interface') }}</button>
		</v-notice>

		<extension-options
			v-if="interfaceId && selectedInterface"
			v-model="options"
			type="interface"
			:options="customOptionsFields"
			:extension="interfaceId"
			show-advanced
		/>
	</div>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, computed } from 'vue';
import { useFieldDetailStore, syncFieldDetailStoreProperty } from '../store/';
import { storeToRefs } from 'pinia';
import ExtensionOptions from '../shared/extension-options.vue';
import { useExtension } from '@/composables/use-extension';

export default defineComponent({
	components: { ExtensionOptions },
	setup() {
		const { t } = useI18n();

		const fieldDetailStore = useFieldDetailStore();

		const interfaceId = syncFieldDetailStoreProperty('field.meta.interface');

		const { field, interfacesForType } = storeToRefs(fieldDetailStore);
		const type = computed(() => field.value.type);

		const selectItems = computed(() => {
			const recommendedInterfacesPerType: { [type: string]: string[] } = {
				string: ['input', 'select-dropdown'],
				text: ['input-rich-text-html'],
				boolean: ['boolean'],
				integer: ['input'],
				bigInteger: ['input'],
				float: ['input'],
				decimal: ['input'],
				timestamp: ['datetime'],
				datetime: ['datetime'],
				date: ['datetime'],
				time: ['datetime'],
				json: ['select-multiple-checkbox', 'tags'],
				uuid: ['input'],
				csv: ['tags'],
			};

			const recommended = recommendedInterfacesPerType[type.value ?? 'alias'] || [];

			const interfaceItems: FancySelectItem[] = interfacesForType.value.map((inter) => {
				const item: FancySelectItem = {
					text: inter.name,
					description: inter.description,
					value: inter.id,
					icon: inter.icon,
				};

				if (recommended.includes(item.value as string)) {
					item.iconRight = 'star';
				}

				return item;
			});

			const recommendedItems: (FancySelectItem | { divider: boolean } | undefined)[] = [];

			const recommendedList = recommended.map((key) => interfaceItems.find((item) => item.value === key));
			if (recommendedList !== undefined) {
				recommendedItems.push(...recommendedList.filter((i) => i));
			}

			if (interfaceItems.length >= 5 && recommended.length > 0) {
				recommendedItems.push({ divider: true });
			}

			const interfaceList = interfaceItems.filter((item) => recommended.includes(item.value as string) === false);
			if (interfaceList !== undefined) {
				recommendedItems.push(...interfaceList.filter((i) => i));
			}

			return recommendedItems;
		});

		const selectedInterface = useExtension('interface', interfaceId);

		const customOptionsFields = computed(() => {
			if (typeof selectedInterface.value?.options === 'function') {
				return selectedInterface.value?.options(fieldDetailStore);
			}

			return null;
		});

		const options = computed({
			get() {
				return fieldDetailStore.field.meta?.options ?? {};
			},
			set(newOptions: Record<string, any>) {
				fieldDetailStore.$patch((state) => {
					state.field.meta = {
						...(state.field.meta ?? {}),
						options: newOptions,
					};
				});
			},
		});

		return { t, selectItems, selectedInterface, interfaceId, customOptionsFields, options };
	},
});
</script>

<style lang="scss" scoped>
.type-title,
.select {
	margin-bottom: 32px;
}

.not-found {
	.spacer {
		flex-grow: 1;
	}

	button {
		text-decoration: underline;
	}
}

.v-notice {
	margin-bottom: 36px;
}
</style>
