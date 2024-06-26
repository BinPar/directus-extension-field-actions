<!-- @see https://github.com/directus/directus/blob/main/app/src/interfaces/input/input.vue -->

<template>
	<div class="action-interface">
		<v-input :model-value="computedCopyValue" :disabled="true" :type="inputType" :placeholder="placeholder" :min="min"
			:max="max" :step="step" v-tooltip="actionTooltip" @click="valueClickAction">
			<template v-if="iconLeft" #prepend>
				<v-icon :name="iconLeft" />
			</template>

			<template v-if="iconRight" #append>
				<v-icon :name="iconRight" />
			</template>
		</v-input>

		<v-button @click.stop="copyValue" v-if="showCopy && isCopySupported" :disabled="!value"
			v-tooltip="value ? `Copy: ${computedCopyValue}` : `Can't copy empty value`" icon secondary xLarge
			:class="copyPosition === 'start' ? '-order-1' : 'order-1'">
			<v-icon name="content_copy" />
		</v-button>

		<!-- TODO: button supports :to=routerLink and :href=custom link. Switch from custom a-tag to those. Use condition: href for full url and "to" for internal links (incomplete url)  -->
		<v-button v-if="showLink" :disabled="!value"
			v-tooltip="value ? `Follow link: ${computedLink}` : `Can't follow empty link`" icon secondary xLarge
			:class="linkPosition === 'start' ? '-order-1' : 'order-1'">
			<a :href="computedLink" :target="openLinkAsNewTab ? '_blank' : '_self'" rel="noopener noreferrer" @click.stop>
				<v-icon name="open_in_new" />
			</a>
		</v-button>
	</div>
</template>

<script setup lang="ts">
import { computed, inject, ref, watch } from 'vue';
import { useClipboard } from '../shared/composable/use-clipboard';
import { usePrefixedValues } from '../shared/composable/use-prefixed-values';
import { useStores } from '@directus/extensions-sdk';
import { render } from 'micromustache';
import slugify from '@sindresorhus/slugify'
import { debounce } from 'lodash';

const props = defineProps({
	value: {
		type: [String, Number],
		default: null,
	},
	field: {
		type: String,
		default: null,
		required: true,
	},
	clickAction: {
		type: String,
		default: 'default',
	},
	showCopy: {
		type: Boolean,
		default: true,
	},
	copyPosition: {
		type: String,
		default: 'end',
	},
	copyPrefix: {
		type: String,
		default: '',
	},
	showLink: {
		type: Boolean,
		default: false,
	},
	linkPosition: {
		type: String,
		default: 'end',
	},
	linkPrefix: {
		type: String,
		default: '',
	},
	placeholder: {
		type: String,
		default: null,
	},
	iconLeft: {
		type: String,
		default: null,
	},
	iconRight: {
		type: String,
		default: null,
	},
	min: {
		type: Number,
		default: undefined,
	},
	max: {
		type: Number,
		default: undefined,
	},
	step: {
		type: Number,
		default: 1,
	},
	relationCollection: {
		type: String,
		default: null,
	},
	template: {
		type: String,
		default: null,
	},
	// global optiopns (independend from this interface)
	type: {
		type: String,
		default: null,
	},
	disabled: {
		type: Boolean,
		default: false,
	},
	openLinkAsNewTab: {
		type: Boolean,
		default: true
	},

});

const emit = defineEmits(['input']);
const values = inject('values', ref<Record<string, any>>({}));

const { isCopySupported, copyToClipboard } = useClipboard();

const { useNotificationsStore } = useStores();
const notificationStore = useNotificationsStore();

watch(values, debounce((values: Record<string, any>) => {
	emitter(values);
}, 200));

function transform(value: string) {
	return slugify(value || '', { separator: '-', preserveCharacters: ['?', '/', '=', ':'] })
}

function emitter(values: Record<string, any>) {
	const valueRender = render(props.template || `${props.value}`, values);
	const newValue = transform(valueRender);

	if (newValue === (props.value || '')) return;

	if (newValue === '') {
		emit('input', 'text');
		return;
	};

	emit('input', newValue)
}

const { computedLink, computedCopyValue } = usePrefixedValues(props);

const inputType = computed(() => {
	if (['bigInteger', 'integer', 'float', 'decimal'].includes(props.type)) return 'number';
	return 'text';
});


async function copyValue() {
	await copyToClipboard(`${computedCopyValue.value}`, notificationStore);
};

// TODO: move in composable (together with display)
function valueClickAction(e: Event) {
	if (props.clickAction === 'copy' && props.disabled && props.value) {
		e.stopPropagation();
		copyValue();
	}

	if (props.clickAction === 'link' && props.disabled && props.value) {
		e.stopPropagation();
		window.open(computedLink.value, '_blank', 'noopener, noreferrer');
	}

	// else go on with the default events
}


// TODO: move in composable (together with display)
const actionTooltip = computed(() => {
	if (props.clickAction === 'copy' && isCopySupported) return 'Copy value';
	if (props.clickAction === 'link') return 'Open link';
	return '';
});

</script>


<style scoped lang="scss">
.action-interface {
	display: flex;
	flex-direction: row;
	align-items: center;

	>div {
		display: inherit;

		&.order-1 {
			order: 1;
			margin-left: 8px;
		}

		&.-order-1 {
			order: -1;
			margin-right: 8px;
		}
	}


}
</style>
