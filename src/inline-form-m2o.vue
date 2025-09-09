<template>
	<v-notice v-if="!readAllowed" type="warning">
		{{ t('interfaces.inline-form-m2o.no-read-permission') }}
	</v-notice>
	<v-notice v-else-if="!relationInfo" type="warning">
		{{ t('relationship_not_setup') }}
	</v-notice>
	<div v-else class="wrapper">
		<!-- Commented out because it wasn't working when creating a new entry createAllowed seem bugged -->
		<!-- <v-notice v-if="!(createAllowed || updateAllowed)" type="warning">
			{{ t('interfaces.inline-form-m2o.no-update-permission') }}
		</v-notice> -->
		<v-form
			:key="currentPrimaryKey"
			v-model="internalEdits"
			:disabled="disabled"
			:loading="loading"
			:show-no-visible-fields="false"
			:initial-values="initialValues"
			:primary-key="currentPrimaryKey"
			:fields="fields"
			:validation-errors="validationErrors"
		/>
	</div>
</template>

<script lang="ts" setup>
import { useApi, useStores } from '@directus/extensions-sdk';
import { computed, ref, toRefs, watch, inject } from 'vue';
import { useI18n } from '@/composables/use-i18n';
import { get, isEmpty, isNil } from 'lodash-es';
import { useRelationM2O } from '@/composables/use-relation-m2o.js';
import { getEndpoint } from '@directus/utils';
import { unexpectedError } from '@/utils/unexpected-error.js';
import { useItemPermissions } from '@/composables/use-permissions';

interface Props {
	value?: string | number | Record<string, any> | null;
	collection: string;
	field: string;
	disabled?: boolean;
	createRelatedItem?: 'withContent' | 'always';
}

const props = withDefaults(defineProps<Props>(), {
	value: null,
	disabled: false,
	createRelatedItem: 'withContent',
});

const emit = defineEmits<{
	(e: 'input', value: Props['value']): void;
}>();

const { collection, field } = toRefs(props);
const { t } = useI18n();
const api = useApi();

const validationErrors = ref<any[]>([]);

const { relationInfo } = useRelationM2O(collection, field);
const currentPrimaryKey = computed<string | number>(() => {
	if (!props.value || !relationInfo.value) return '+';

	if (typeof props.value === 'number' || typeof props.value === 'string') {
		return props.value;
	}

	const primaryKey = get(props.value, relationInfo.value.relatedPrimaryKeyField.field, '+');
	return primaryKey;
});
const isNew = computed(() => currentPrimaryKey.value === '+');

const { internalEdits, loading, initialValues, fetchItem, isInitializing } = useItem();
const { fields: fieldsWithPermissions } = useItemPermissions(
	computed(() => relationInfo.value?.relatedCollection.collection ?? ''),
	currentPrimaryKey,
	isNew,
);

const { usePermissionsStore } = useStores();
const { hasPermission } = usePermissionsStore();
const readAllowed = computed(() => {
	if (!relationInfo.value) return false;
	return hasPermission(relationInfo.value.relatedCollection.collection, 'read');
});

// don't show circular field
const fields = computed(() =>
	!isNil(relationInfo.value?.relation.meta?.one_field)
		? fieldsWithPermissions.value.filter(({ field }) => field !== relationInfo.value?.relation.meta?.one_field)
		: fieldsWithPermissions.value,
);

watch(
	internalEdits,
	() => {
		if (!relationInfo.value) return;

		// Don't emit input events during initialization to prevent "unsaved changes" popup
		if (isInitializing.value) {
			return;
		}

		if (!isEmpty(internalEdits.value) || (isNew.value && props.createRelatedItem === 'always')) {
			const item = internalEdits.value;

			if (!isNew.value) {
				item[relationInfo.value.relatedPrimaryKeyField.field] = currentPrimaryKey.value;
			}

			emit('input', item);
		}
	},
	{ deep: true, immediate: true },
);

// watch for a discard (value will be changed back to it's initial key value)
// refetch data if this is not a new item
watch(
	() => props.value,
	() => {
		if (!relationInfo.value) return;

		if (get(initialValues.value, relationInfo.value.relatedPrimaryKeyField.field) === props.value && !isNew.value) {
			fetchItem();
		}
	},
);

// Watch for changes in initialValues and update internalEdits accordingly
// This ensures the form displays the correct values when switching between versions
watch(
	() => initialValues.value,
	(newInitialValues) => {
		if (newInitialValues && !isNew.value) {
			// Set initialization flag to prevent emit during data loading
			isInitializing.value = true;

			// Update internalEdits to reflect the new initial values
			internalEdits.value = { ...newInitialValues };

			// Clear initialization flag after a short delay to allow the form to settle
			setTimeout(() => {
				isInitializing.value = false;
			}, 100);
		}
	},
	{ deep: true, immediate: true },
);

function useItem() {
	const internalEdits = ref<Record<string, any>>({});
	const loading = ref(false);
	const initialValues = ref<Record<string, any> | null>(null);
	const isInitializing = ref(false); // Track if we're in the middle of loading initial data
	const { useNotificationsStore } = useStores();
	const notificationStore = useNotificationsStore();

	watch(
		[currentPrimaryKey, isNew, relationInfo],
		() => {
			if (!isNew.value) fetchItem();
		},
		{ immediate: true },
	);

	return { internalEdits, loading, initialValues, fetchItem, getVersionContext, isInitializing };

	/**
	 * Gets the current version context from multiple sources
	 * This ensures that related items are fetched with the correct version/draft context
	 */
	function getVersionContext(): { version?: string; draft?: string } {
		const context: { version?: string; draft?: string } = {};

		try {
			// Method 1: Try to inject version context from parent form
			try {
				const parentVersion = inject('version', null);
				const parentDraft = inject('draft', null);
				if (parentVersion) context.version = parentVersion;
				if (parentDraft) context.draft = parentDraft;
			} catch (e) {
				// Injection failed, continue with other methods
			}

			// Method 2: Check URL parameters
			const urlParams = new URLSearchParams(window.location.search);
			if (urlParams.has('version') && !context.version) {
				context.version = urlParams.get('version') || undefined;
			}
			if (urlParams.has('draft') && !context.draft) {
				context.draft = urlParams.get('draft') || undefined;
			}

			// Method 3: Check route path for version/draft context
			const route = window.location.pathname;
			if (route.includes('/versions/') || route.includes('/draft/')) {
				const routeMatch = route.match(/\/(versions|draft)\/([^/]+)/);
				if (routeMatch) {
					if (routeMatch[1] === 'versions' && !context.version) {
						context.version = routeMatch[2];
					} else if (routeMatch[1] === 'draft' && !context.draft) {
						context.draft = routeMatch[2];
					}
				}
			}

			// Method 4: Check for hash-based version context
			const hash = window.location.hash;
			if (hash) {
				const hashParams = new URLSearchParams(hash.substring(1));
				if (hashParams.has('version') && !context.version) {
					context.version = hashParams.get('version') || undefined;
				}
				if (hashParams.has('draft') && !context.draft) {
					context.draft = hashParams.get('draft') || undefined;
				}
			}

			// Method 5: Check if we're in a draft context by looking at the current page
			// This is a fallback for when other methods don't work
			if (!context.version && !context.draft) {
				const currentUrl = window.location.href;
				if (currentUrl.includes('draft=true') || currentUrl.includes('draft=1')) {
					context.draft = 'true';
				}
			}
		} catch (e) {
			// If any error occurs, return empty context
			// eslint-disable-next-line no-console
			console.warn('Failed to get version context:', e);
		}

		return context;
	}

	async function fetchItem() {
		if (!currentPrimaryKey.value || !relationInfo.value) return;

		loading.value = true;

		const versionContext = getVersionContext();

		try {
			let response;

			// Method 1: Try to fetch through the parent item's version context
			// This ensures we get the related item data in the same version context as the parent
			if (versionContext.version || versionContext.draft) {
				// Get the parent collection and primary key from the URL
				const urlMatch = window.location.pathname.match(/\/admin\/content\/([^/]+)\/([^/?]+)/);
				if (urlMatch && urlMatch[1] && urlMatch[2]) {
					const parentCollection = urlMatch[1];
					const parentPrimaryKey = urlMatch[2];

					// Try to fetch the parent item with version context and include the related field
					const parentEndpoint = getEndpoint(parentCollection);
					const parentUrl = parentCollection.startsWith('directus_')
						? `${parentEndpoint}/${parentPrimaryKey}`
						: `${parentEndpoint}/${encodeURIComponent(parentPrimaryKey)}`;

					const parentParams: Record<string, any> = {
						fields: `*,${props.field}.*`,
					};

					if (versionContext.version) {
						parentParams.version = versionContext.version;
						parentParams.versionRaw = true; // CRITICAL: Get versioned format for M2M fields
					}
					if (versionContext.draft) {
						parentParams.draft = versionContext.draft;
					}

					const parentResponse = await api.get(parentUrl, { params: parentParams });

					// Extract the related item data from the parent response
					const relatedData = get(parentResponse.data.data, props.field);
					if (relatedData && relatedData.id === currentPrimaryKey.value) {
						response = { data: { data: relatedData } };
					}
				}
			}

			// Method 2: Fallback to direct API call if Method 1 didn't work
			if (!response) {
				const baseEndpoint = getEndpoint(relationInfo.value.relatedCollection.collection);
				const endpoint = relationInfo.value.relatedCollection.collection.startsWith('directus_')
					? `${baseEndpoint}/${currentPrimaryKey.value}`
					: `${baseEndpoint}/${encodeURIComponent(currentPrimaryKey.value)}`;

				const params: Record<string, any> = { fields: '*' };

				// Try different parameter names for version context
				if (versionContext.version) {
					params.version = versionContext.version;
					params.version_id = versionContext.version;
					params.version_key = versionContext.version;
				}
				if (versionContext.draft) {
					params.draft = versionContext.draft;
					params.draft_key = versionContext.draft;
				}

				response = await api.get(endpoint, { params });
			}

			initialValues.value = response.data.data;
		} catch (err: any) {
			// eslint-disable-next-line no-console
			console.error('API request failed:', err);
			unexpectedError(err, notificationStore, t);
		} finally {
			loading.value = false;
			// Don't clear internalEdits here as we want to keep the loaded data
			// internalEdits.value = {};
		}
	}
}
</script>

<style lang="scss" scoped>
.wrapper {
	& > div + div {
		margin-top: var(--form-vertical-gap);
	}
}
</style>
