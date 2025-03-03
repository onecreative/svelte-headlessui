<script  context="module">"use strict";
</script>

<script >import { useMenuContext, MenuStates } from "./Menu.svelte";
import { useId } from "../../hooks/use-id";
import { Keys } from "../../utils/keyboard";
import { Focus } from "../../utils/calculate-active-index";
import { treeWalker } from "../../hooks/use-tree-walker";
import { State, useOpenClosed } from "../../internal/open-closed";
import { tick } from "svelte";
import Render from "../../utils/Render.svelte";
import { forwardEventsBuilder } from "../../internal/forwardEventsBuilder";
import { get_current_component } from "svelte/internal";
import { Features } from "../../types";
import { computePosition, flip, shift, offset, arrow } from '@floating-ui/dom';
export let as = "div";
export let use = [];
/***** Events *****/
const forwardEvents = forwardEventsBuilder(get_current_component());
/***** Component *****/
const api = useMenuContext("MenuItems");
const id = `headlessui-menu-items-${useId()}`;
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
    container: $itemsStore,
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
async function handleKeyDown(e) {
    if (searchDebounce)
        clearTimeout(searchDebounce);
    let event = e;
    switch (event.key) {
        // Ref: https://www.w3.org/TR/wai-aria-practices-1.2/#keyboard-interaction-12
        case Keys.Space:
            if ($api.searchQuery !== "") {
                event.preventDefault();
                event.stopPropagation();
                return $api.search(event.key);
            }
        // When in type ahead mode, fallthrough
        case Keys.Enter:
            event.preventDefault();
            event.stopPropagation();
            if ($api.activeItemIndex !== null) {
                let { id } = $api.items[$api.activeItemIndex];
                document.getElementById(id)?.click();
            }
            $api.closeMenu();
            await tick();
            $buttonStore?.focus({ preventScroll: true });
            break;
        case Keys.ArrowDown:
            event.preventDefault();
            event.stopPropagation();
            return $api.goToItem(Focus.Next);
        case Keys.ArrowUp:
            event.preventDefault();
            event.stopPropagation();
            return $api.goToItem(Focus.Previous);
        case Keys.Home:
        case Keys.PageUp:
            event.preventDefault();
            event.stopPropagation();
            return $api.goToItem(Focus.First);
        case Keys.End:
        case Keys.PageDown:
            event.preventDefault();
            event.stopPropagation();
            return $api.goToItem(Focus.Last);
        case Keys.Escape:
            event.preventDefault();
            event.stopPropagation();
            $api.closeMenu();
            await tick();
            $buttonStore?.focus({ preventScroll: true });
            break;
        case Keys.Tab:
            event.preventDefault();
            event.stopPropagation();
            break;
        default:
            if (event.key.length === 1) {
                $api.search(event.key);
                searchDebounce = setTimeout(() => $api.clearSearch(), 350);
            }
            break;
    }
}
function handleKeyUp(e) {
    let event = e;
    switch (event.key) {
        case Keys.Space:
            // Required for firefox, event.preventDefault() in handleKeyDown for
            // the Space key doesn't cancel the handleKeyUp, which in turn
            // triggers a *click*.
            event.preventDefault();
            break;
    }
}
$: {
    //position the submenu
    if ($buttonStore && $itemsStore) {
        computePosition($buttonStore, $itemsStore, {
            placement: 'bottom',
            middleware: [
                offset(30),
                flip(),
                shift({padding: 5}),
                arrow({
                    element: $arrowStore,
                    padding: 5
                }),
            ],
        }).then(({x, y, placement, middlewareData}) => {
            if ($itemsStore) {
                Object.assign($itemsStore.style, {
                    left: `${x}px`,
                    top: `${y}px`,
                });
            }
            // Accessing the data
            const {x: arrowX, y: arrowY} = middlewareData.arrow;

            const staticSide = {
                top: 'bottom',
                right: 'left',
                bottom: 'top',
                left: 'right',
            }[placement.split('-')[0]];

            if ($arrowStore) {
                Object.assign($arrowStore.style, {
                    left: arrowX != null ? `${arrowX}px` : '',
                    top: arrowY != null ? `${arrowY}px` : '',
                    right: '',
                    bottom: '',
                    [staticSide]: '-14px',
                });
            }
        });
    }
}

$: propsWeControl = {
    "aria-activedescendant": $api.activeItemIndex === null
        ? undefined
        : $api.items[$api.activeItemIndex]?.id,
    "aria-labelledby": $buttonStore?.id,
    id,
    role: "menu",
    tabIndex: 0,
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
  bind:el={$itemsStore}
  name={"MenuItems"}
  on:keydown={handleKeyDown}
  on:keyup={handleKeyUp}
  {visible}
  features={Features.RenderStrategy | Features.Static}
>
  <slot {...slotProps} />
</Render>
