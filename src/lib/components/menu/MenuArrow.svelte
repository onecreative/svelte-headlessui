<script context="module">
	"use strict";
</script>

<script>
	import { useMenuContext, MenuStates } from "./Menu.svelte";
	import { useId } from "../../hooks/use-id";
	import { treeWalker } from "../../hooks/use-tree-walker";
	import { State, useOpenClosed } from "../../internal/open-closed";
	import { tick } from "svelte";
	import Render from "../../utils/Render.svelte";
	import { forwardEventsBuilder } from "../../internal/forwardEventsBuilder";
	import { get_current_component } from "svelte/internal";
	import { Features } from "../../types";

	export let as = "div";
	export let use = [];

	/***** Events *****/
	const forwardEvents = forwardEventsBuilder(get_current_component());

	/***** Component *****/
	const api = useMenuContext("MenuArrow");
	const id = `headlessui-menu-arrow-${useId()}`;
	let searchDebounce = null;
	$: buttonStore = $api.buttonStore;
	$: itemsStore = $api.itemsStore;
	$: arrowStore = $api.arrowStore;
	let openClosedState = useOpenClosed();
	$: visible =
		openClosedState !== undefined
			? $openClosedState === State.Open
			: $api.menuState === MenuStates.Open;
	$: treeWalker({
		container: $arrowStore,
		enabled: $api.menuState === MenuStates.Open,
		accept(node) {
			if (node.getAttribute("role") === "menuitem")
				return NodeFilter.FILTER_REJECT;
			if (node.hasAttribute("role"))
				return NodeFilter.FILTER_SKIP;
			return NodeFilter.FILTER_ACCEPT;
		},
		walk(node) {
			node.setAttribute("role", "none");
		},
	});

	$: propsWeControl = {
		id,
		tabIndex: -1,
	};
	$: slotProps = {
		open: $api.menuState === MenuStates.Open,
	};
</script>

<Render
  {...{ ...$$restProps, ...propsWeControl }}
  {as}
  {slotProps}
  use={[...use, forwardEvents]}
  bind:el={$arrowStore}
  name={"MenuArrow"}
  {visible}
  features={Features.RenderStrategy | Features.Static}
>
  <slot {...slotProps} />
</Render>
