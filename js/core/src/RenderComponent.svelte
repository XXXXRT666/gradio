<svelte:options immutable={true} />

<script lang="ts">
	import type { ComponentMeta, ThemeMode } from "./types";
	import type { SvelteComponent, ComponentType } from "svelte";
	import { translate_if_needed, extractI18nKey } from "./i18n";
	// @ts-ignore
	import { bind, binding_callbacks } from "svelte/internal";

	export let root: string;
	export let component: ComponentMeta["component"];
	export let target: HTMLElement;
	export let theme_mode: ThemeMode;
	export let instance: ComponentMeta["instance"];
	export let value: any;
	// export let gradio: Gradio;
	export let elem_id: string;
	export let elem_classes: string[];
	export let _id: number;
	export let visible: boolean;

	const s = (id: number, p: string, v: any): CustomEvent =>
		new CustomEvent("prop_change", { detail: { id, prop: p, value: v } });

	function wrap(
		component: ComponentType<SvelteComponent>
	): ComponentType<SvelteComponent> {
		const ProxiedMyClass = new Proxy(component, {
			construct(_target, args: Record<string, any>[]) {
				//@ts-ignore
				const instance = new _target(...args);
				const props = Object.keys(instance.$$.props);

				function report(props: string) {
					return function (propargs: any) {
						if (!target) return;
						const ev = s(_id, props, propargs);
						target.dispatchEvent(ev);
					};
				}
				props.forEach((v) => {
					binding_callbacks.push(() => bind(instance, v, report(v)));
				});

				return instance;
			}
		});

		return ProxiedMyClass;
	}

	let _component = wrap(component);

	const supported_props = [
		"description",
		"info",
		"title",
		"placeholder",
		"value",
		"label",
		"choices"
	];

	function translate_prop(obj: SvelteRestProps): void {
		for (const key in obj) {
			if (supported_props.includes(key as string)) {
				if (key === "choices" && Array.isArray(obj[key])) {
					obj[key] = obj[key].map((choice: any) => {
						return [translate_if_needed(choice[0]), extractI18nKey(choice[1])];
					});
				} else {
					obj[key] = translate_if_needed(obj[key]);
				}
			}
		}
	}

	$: translate_prop($$restProps);
	$: value = translate_if_needed(value);
</script>

{#if visible}
<svelte:component
	this={_component}
	bind:this={instance}
	bind:value
	on:prop_change
	{elem_id}
	{elem_classes}
	{target}
	{...$$restProps}
	{theme_mode}
	{root}
>
	<slot />
</svelte:component>
{/if}
