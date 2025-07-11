<script>
  import { getContext, onMount } from "svelte";
  import { rippleFocus } from "#standard/action/animate/composable";
  import { TJSDocument } from "#runtime/svelte/store/fvtt/document";

  import { TJSInput } from "#standard/component/form";
  import { createFilterQuery } from "~/src/filters/itemFilterQuery";
  import { toggleBookmark, localize } from "~/src/helpers/util";
  import { SYSTEM_ID, SYSTEM_CODE } from "~/src/helpers/constants";
  import ProseMirror from "~/src/components/molecules/ProseMirror.svelte";
  import ScrollingContainer from "~/src/helpers/svelte-components/ScrollingContainer.svelte";
  import InventoryRow from "~/src/components/molecules/InventoryRow.svelte";
  import Badge from "~/src/components/atoms/Badge.svelte";

  const Actor = getContext("#doc");
  const doc = new TJSDocument($Actor);
  const typeSearch = createFilterQuery("system.favourite");
  typeSearch.set(true)
  const input = {
    store: typeSearch,
    efx: rippleFocus(),
    placeholder: "by Name",
    type: "search",
    id: "search",
  };

  /** @type {import('@typhonjs-fvtt/runtime/svelte/store').DynMapReducer<string, Item>} */
  const wildcard = doc.embedded.create(Item, {
    name: "wildcard",
    filters: [typeSearch],
    sort: (a, b) => a.name.localeCompare(b.name),
  });

  let combat;
  
  function onCombatUpdate() {
    combat = game.combat;
  }

  onMount(() => {
    // Initial combat state
    combat = game.combat;
    
    // Subscribe to combat updates @why? – 
    Hooks.on('createCombat', onCombatUpdate);
    Hooks.on('deleteCombat', onCombatUpdate);
    Hooks.on('updateCombat', onCombatUpdate);
    
    return () => {
      // Cleanup hooks on component destroy
      Hooks.off('createCombat', onCombatUpdate);
      Hooks.off('deleteCombat', onCombatUpdate);
      Hooks.off('updateCombat', onCombatUpdate);
    };
  });

  function editItem(item) {
    item.sheet.render(true);

    game.system.log.d("editItem");
    game.system.log.d(item);
  }

  function addQuantity(item) {
    game.system.log.d("addQuantity");
    game.system.log.d(item);

    const quantity = item.system.quantity + 1;
    item.update({ system: { quantity: quantity } });
  }

  function removeQuantity(item) {
    const quantity = item.system.quantity - 1;
    item.update({ system: { quantity: quantity } });
  }

  function duplicateItem(item) {
    const itemData = item.toObject();
    delete itemData._id;
    $Actor.sheet._onDropItemCreate(itemData);
  }

  function deleteItem(index, item) {
    let okToDelete = true;
    if (game.settings.get(SYSTEM_ID, "confirmBeforeDeletingActorItem")) {
      okToDelete = confirm(localize("Types.Actor.Inventory.confirmDeleteItem"));
    }
    if (okToDelete) {
      item.delete();
    }
  }

  function roll(item) {
    game.system.log.d("roll");
    game.system.log.d(item);
  }
  function useItem(item) {
    switch (item.type) {
      case 'action':
        new CONFIG.FFXIV.RollCalcActor({ actor: $Actor}).ability(item.type, item);
        break;

      case 'trait':
        new CONFIG.FFXIV.RollCalcActor({ actor: $Actor}).ability(item.type, item);
        break;

      case 'equipment':
        new CONFIG.FFXIV.RollCalcActor({ actor: $Actor }).equipment(item);
        break;

      case 'effect':
        ui.notifications.warn("Effects cannot be used directly");
        break;

      case 'job':
        ui.notifications.warn("Jobs cannot be used directly");
        break;

      case 'limitbreak':
        new CONFIG.FFXIV.RollCalcActor({ actor: $Actor}).ability(item.type, item);
        break;

      default:
        console.warn(`Unhandled item type: ${item.type}`);
        new CONFIG.FFXIV.RollCalcActor({ actor: $Actor }).send();
    }
  }
  function toggleLock(event) {
    game.system.log.d("a");
    event.stopPropagation();
    event.preventDefault();
    $doc.update(
      {
        ["system.inventoryLocked"]: !$doc.system.inventoryLocked,
      },
      {
        diff: true,
        diffData: true,
        diffSystem: true,
      },
    );
  }

  function showItemSheet(item) {
    item.sheet.render(true);
  }

  $: items = [...$wildcard];
  $: lockCSS = $doc.system.inventoryLocked ? "lock" : "lock-open";
  $: faLockCSS = $doc.system.inventoryLocked ? "fa-lock negative" : "fa-lock-open positive";
  
  const badgeType = (item) => {
    return item.system.uses >= item.system.limitation ? 'danger' : 'success';
  }
  
  const remaining = (item) => {
    return item.system.hasLimitation ? 
      parseInt(item.system.limitation || 0) - parseInt(item.system.uses || 0) : 10;
  } 
</script>

<template lang="pug">
.favourites({...$$restProps})
  +if("$doc.system.hasFavouriteItems()")
    h2.font-cinzel {localize("Tabs.Favourites")}
    div.pa-xs
      table.borderless.low-contrast
        tr
          th.img.shrink(scope="col")
          th.left.expand(scope="col") {localize("Name")}
          +if("combat")
            th
          th.shrink(scope="col")
            
        +each("items as item, index")
          tr
            td.img(on:click="{useItem(item)}" role="button")
              img.icon(src="{item.img}" alt="{item.name}"  )
            td.left.clip
              a.ml-sm.stealth.link(on:click="{showItemSheet(item)}" class="{item.system.isMagic ? 'pulse' : ''}" role="button") {item.name}
            +if("combat")
              td.left
                +if("item.system.hasLimitation")
                  Badge(type!="{badgeType(item)}") {remaining(item)}
            td
              button.stealth(on:click="{toggleBookmark(item)}") 
                i.fa-bookmark(class="{item.system.favourite === true ? 'fa-solid' : 'fa-regular'}" )
</template>

<style lang="sass">
@use '../../../styles/_mixins' as mixins

.favourites
  +mixins.inset(0.5rem, 0 0 5px rgba(165,0,0,1) inset)
  width: 100%

.fa-bookmark
  color: var(--color-highlight) !important
  cursor: pointer
  &.row
    color: rgba(100, 0, 100, 1)

.pulse
  +mixins.pulse

.actions
  margin-left: 0.5rem
  margin-right: 0
  justify-content: right
  :not(:last-child)
    margin-right: 2px

.clickable
  max-height: 1.3rem
  line-height: 1.3rem
  background: rgba(255, 255, 255, 0.2)

.clip
  text-overflow: ellipsis
  overflow: hidden
  height: 2rem
  max-height: 2rem
  display: block
</style>
