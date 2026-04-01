<script lang="ts">
  import { slide } from 'svelte/transition';
  import { forumsData, type IForum } from './forums';
  import { onMount } from 'svelte';
  import Spinner from './Spinner.svelte';

  let forums: IForum[][] = $state([]);
  let isLoading: boolean = $state(true);
  let colsPerRow = $state(3);
  let winW = $state(0);
  $effect(() => {
    colsPerRow = winW <= 520 ? 1 : winW <= 720 ? 2 : 3;
  });

  let selectedForum: IForum | null = $state(null);
  let selectedSubForums: IForum[] = $state([]);
  let searchText: string = $state('');

  const baseCategories = $derived(forums.flat().filter((x) => x.node_type_id === 'Category')!);
  const forumsNoCategory = $derived(forums.flat().filter((x) => x.parent_node_id === 0 && x.node_type_id !== 'Category')!);
  const selected: IForum | null = $derived(selectedSubForums[selectedSubForums.length - 1] ?? selectedForum);
  const crumbs = $derived([selectedForum, ...selectedSubForums].filter(Boolean));

  const categories = $derived(
    baseCategories.concat(forumsNoCategory.length > 0
      ? [{
        forum_id: -1,
        forum_title: 'Без категории',
        forum_description: '',
        forum_thread_count: 0,
        forum_post_count: 0,
        parent_node_id: 0,
        node_type_id: 'Category',
        icon_content: '',
        active_icon_content: '',
        forum_rules_thread_id: null,
        forum_is_followed: false,
        links: {},
        permissions: {}
      } as IForum]
      : [])
  );
  const allForumsByParentId = $derived(
    Object.fromEntries(
      forums
        .filter((group) => group.length > 0)
        .map((group) => [group[0].parent_node_id, group])
        .concat(forumsNoCategory.length > 0 ? [[-1, forumsNoCategory]] : [])
    ) as Record<number, IForum[]>
  );


  async function loadForums() {
    await new Promise((res) => setTimeout(res, 1000));
    forums = forumsData.data;
  }
  onMount(async () => {
    await loadForums();
    isLoading = false;
  });

  const findForumById = (forumId: number) => forums.flat().find((x) => x.forum_id === forumId);
  const selectForum = (forumId: number) => {
    if (selectedForum?.forum_id === forumId) {
      if (selectedSubForums.length > 0) {
        selectedSubForums = [];
      } else {
        selectedForum = null;
      }
    } else {
      selectedForum = findForumById(forumId)!;
      selectedSubForums = [];
    }
  };
  const selectSubForum = (subForumId: number, deep: number) => {
    const hide = selectedSubForums.some((x) => x.forum_id === subForumId);
    if (hide) {
      if (deep === 1) {
        if (selectedSubForums.length === 1) selectedSubForums = [];
        else selectedSubForums = selectedSubForums.slice(0, 1);
      } else if (deep === 2 && selectedSubForums.length > 0) {
        selectedSubForums = selectedSubForums.slice(0, 1);
      }
      return;
    }
    if (deep === 1) {
      selectedSubForums = [findForumById(subForumId)!];
    } else if (deep === 2 && selectedSubForums.length > 0) {
      selectedSubForums = [selectedSubForums[0], findForumById(subForumId)!];
    }
  };

  // фильтр по поиску
  const norm = (s: string) => (s ?? '').toLowerCase().trim();
  const match = (f: IForum, q: string) => norm(f.forum_title).includes(q) || norm(f.forum_description).includes(q);
  const filteredData: () => { result: Record<number, IForum[]>; keep: Set<number> | null } = $derived(() => {
    const q = norm(searchText);
    if (!q) {
      return { result: allForumsByParentId, keep: null };
    }

    const flat = forums.flat();
    const matched = new Set<number>();
    const displayParentId = (x: IForum) => (x.parent_node_id === 0 && x.node_type_id !== 'Category' ? -1 : x.parent_node_id);
    const byId = new Map(flat.map((x) => [x.forum_id, x] as const));
    const parentOf = new Map(flat.map((x) => [x.forum_id, displayParentId(x)] as const));

    // все по критерию
    for (const f of flat) {
      if (match(f, q)) matched.add(f.forum_id);
    }

    // родители
    const keep = new Set<number>(matched);
    for (const id of matched) {
      for (let cur = id, guard = new Set<number>(); !guard.has(cur); ) {
        guard.add(cur);
        const pid = parentOf.get(cur) ?? 0;
        if (!pid) break;
        keep.add(pid);
        cur = pid;
      }
    }

    // детишек на остров
    for (const id of matched) {
      if (byId.get(id)?.node_type_id === 'Category') {
        for (const ch of allForumsByParentId[id] ?? []) {
          keep.add(ch.forum_id);
        }
      }
    }

    const result: Record<number, IForum[]> = {};
    for (const [pidStr, list] of Object.entries(allForumsByParentId)) {
      const filtered = (list ?? []).filter((x) => keep.has(x.forum_id));
      if (filtered.length) result[+pidStr] = filtered;
    }

    return { result, keep };
  });

  const forumsByParentIdFiltered = $derived(() => filteredData().result);

  // чистим selectedForum если не попадает по критерию
  $effect(() => {
    const keep = filteredData().keep;
    if (!keep) return;
    if (selectedForum && !keep.has(selectedForum.forum_id)) selectedForum = null;
    if (selectedSubForums.length) {
      const next = selectedSubForums.filter((x) => keep.has(x.forum_id));
      if (next.length !== selectedSubForums.length) {
        selectedSubForums = next;
      }
    }
  });
  const selectedForumDescription = $derived(crumbs.reverse().filter((x) => !!x.forum_description)[0]?.forum_description);
</script>

<svelte:window bind:innerWidth={winW} />
<div class="container" >
  <div class="titleContainer">
    <div class="titleWrap">
      <span class="title">Выбор раздела</span>
      <span class="description">Выберите раздел для создания темы</span>
    </div>
  </div>
  <div class="contentContainer">
    <div class="content" class:footerShown={selected}>
      <div class="searchContainer">
        <input type="text" bind:value={searchText} placeholder="Поиск..." />
        {#if searchText}
          <button type="button" class="clearBtn" onclick={() => (searchText = '')} aria-label="Очистить"></button>
        {/if}
      </div>
      {#if isLoading}
        <div class="loadingContainer">
          <Spinner type="medium" variant="logoForum" />
        </div>
      {:else}
        <div class="forums">
          {#each categories as category}
            {@const items = forumsByParentIdFiltered()[category.forum_id] ?? []}
            {@const selectedIndex = selectedForum ? items.findIndex((x) => x.forum_id === selectedForum?.forum_id) : -1}

            {@const selectedRow = selectedIndex !== -1 ? Math.floor(selectedIndex / colsPerRow) : -1}

            {#if items.length > 0}
              <div class="forumContainer">
                <div class="title">{category.forum_title}</div>

                <div class="forumItems">
                  {#each items as item, i}
                    {@const isSelected = selectedForum?.forum_id === item.forum_id}
                    <div class:isSelected class:current={isSelected} class:isActive={isSelected && selectedSubForums.length === 0} class="forumItem nodeTitle" role="button" tabindex="0" onclick={() => selectForum(item.forum_id)} onkeydown={(e) => (e.key === 'Enter' || e.key === ' ') && selectForum(item.forum_id)}>
                      <div class="forumInfo">
                        <div class="forumIcon NodeSvgIcon nodeIcon{item.forum_id}"></div>
                        <b class="title">{item.forum_title}</b>
                      </div>

                      {#if forumsByParentIdFiltered()[item.forum_id]}
                        <span class="hasChildren"></span>
                      {/if}
                    </div>

                    {@const isRowEnd = (i + 1) % colsPerRow === 0 || i === items.length - 1}
                    {@const currentRow = Math.floor(i / colsPerRow)}
                    {@const hasChildren = selectedForum && forumsByParentIdFiltered()[selectedForum.forum_id]}

                    {#if selectedForum && isRowEnd && currentRow === selectedRow && hasChildren}
                      <div class="subBlock" transition:slide={{ duration: 200 }}>
                        <div class="subTitle">{selectedForum.forum_title}: подразделы</div>
                        <div class="subForumItems">
                          {#each forumsByParentIdFiltered()[selectedForum.forum_id] as sub (sub.forum_id)}
                            {@const hasChildren = forumsByParentIdFiltered()[sub.forum_id]}
                            {@const isSelected = selectedSubForums.some((x) => x.forum_id === sub.forum_id)}
                            <div class="subForumContainer">
                              <div class="subForumItem" class:noChildren={!hasChildren} class:isSelected class:isActive={isSelected && selectedSubForums.length === 1} onclick={() => selectSubForum(sub.forum_id, 1)} onkeydown={(e) => (e.key === 'Enter' || e.key === ' ') && selectSubForum(sub.forum_id, 1)} tabindex="0" role="button">
                                <div class="subForumName">
                                  <div class="forumIcon NodeSvgIcon nodeIcon{selectedForum.forum_id}" class:isActive={isSelected && selectedSubForums.length === 1}></div>
                                  <div class="name">{sub.forum_title}</div>
                                </div>
                                {#if hasChildren}
                                  <span class="hasChildren"></span>
                                {/if}
                              </div>
                              {#if hasChildren}
                                {#if isSelected}
                                  <div class="subForumChildren" transition:slide={{ duration: 300 }}>
                                    {#each forumsByParentIdFiltered()[sub.forum_id] as sub2}
                                      {@const isActive = selectedSubForums.some((x) => x.forum_id === sub2.forum_id)}
                                      <div class="subForumChild" class:isActive onclick={() => selectSubForum(sub2.forum_id, 2)} onkeydown={(e) => (e.key === 'Enter' || e.key === ' ') && selectSubForum(sub2.forum_id, 2)} tabindex="0" role="button">{sub2.forum_title}</div>
                                    {/each}
                                  </div>
                                {:else}
                                  <!-- Сохраняем ширину блока в закрытом виде, чтобы анимация не дергалась, атопиздец !-->
                                  <div class="subForumChildren hidden">
                                    {#each forumsByParentIdFiltered()[sub.forum_id] as sub2}
                                      <div class="subForumChild">{sub2.forum_title}</div>
                                    {/each}
                                  </div>
                                {/if}
                              {/if}
                            </div>
                          {/each}
                        </div>
                      </div>
                    {/if}
                  {/each}
                </div>
              </div>
            {/if}
          {/each}
        </div>
        {#if filteredData().keep?.size === 0}
          <div class="nothingFoundContainer">Ничего не найдено</div>
        {/if}
      {/if}
    </div>
  </div>
  {#if selected}
    <div class="footerContainer">
      <div class="footerBlock">
        <div class="footerLeft nodeTitle current">
          <div class="selectedForumIcon forumIcon NodeSvgIcon nodeIcon{selectedForum?.forum_id}"></div>
          <div class="selectedForumInfo">
              <div class="crumbs">
                  {#each crumbs as crumb, i}
                      {@const isLast = i === crumbs.length - 1}
                      <div class="crumb">{isLast ? crumb?.forum_title : crumb?.forum_title}</div>
                      {#if !isLast}
                          <div class="sep">/</div>
                      {/if}
                  {/each}
              </div>
              {#if selectedForumDescription}
                  <div class="forumDesc">{selectedForumDescription}</div>
              {/if}
          </div>
        </div>
        <div class="forumActions">
          {#if selected.forum_rules_thread_id}
            <a class="button" href="threads/{selected.forum_rules_thread_id}/">Правила раздела</a>
          {/if}
          <a class="button createThreadButton" class:disabled={!selected.permissions?.create_thread} href="forums/{selected.forum_id}/create-thread">
            <span class="text">{selected.permissions?.create_thread ? 'Создать тему' : 'Нельзя создать тему'}</span>
          </a>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
    .container {
        position: relative;
        background-color: var(--primaryDarker);
        border-radius: 12px;
    }
    .titleContainer {
        border-radius: 12px 12px 0 0;
        padding: 16px 20px;
        background-image: url(https://lztcdn.com/files/ed0e59d5-a779-4213-b0ce-4c423d76058b.webp);
    }

    .titleContainer .titleWrap {
        display: flex;
        flex-direction: column;
        gap: 6px;
    }

    .titleContainer .titleWrap .title {
        font-size: 16px;
        font-weight: bold;
        line-height: 24px;
    }

    .titleContainer .titleWrap .description {
        line-height: 20px;
        color: #949494;
    }

    .nothingFoundContainer, .loadingContainer {
        height: 50vh;
        display: flex;
        align-items: center;
        justify-content: center;
        color: #949494;
    }

    .footerBlock {
        padding: 16px;
        display: flex;
        flex-wrap: nowrap;
        gap: 20px;
        justify-content: space-between;
        align-items: center;
        z-index: 10;
        overflow: hidden;
        background-color: var(--contentBackground);
        border-radius: 0 0 8px 8px;
        box-shadow: 0 -3px 0 0 #242424;
    }
    @media (max-width: 800px) {
        .footerBlock {
            flex-wrap: wrap;
        }
    }
    .selectedForumInfo {
        display: grid;
        gap: 2px;
    }

    .footerBlock .forumActions {
        flex-shrink: 0;
        display: flex;
        gap: 10px;
        justify-content: center;
    }
    .footerBlock .forumDesc {
        color: #949494;
    }
    .footerBlock .crumbs {
        font-size: 13px;
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        gap: 4px;
        min-width: 0;
    }

    .footerContainer {
        background-color: var(--primaryDarker);
        padding: 0 4px 4px 4px;
        border-radius: 0 0 12px 12px;
        position: sticky;
        bottom: 0;
        z-index: 5555;
    }

    .createThreadButton {
        text-wrap: nowrap;
        position: relative;
        display: inline-flex;
        align-items: center;
        gap: 6px;
        padding: 10px 16px;
        color: #f5f5f5;
        background: linear-gradient(88deg, #1c6946 0.29%, #329c6c 51.58%, #1d8254 100%);
        border-radius: 8px;
        overflow: hidden;
        user-select: none;
    }
    .createThreadButton:not(.disabled)::after {
        content: '';
        position: absolute;
        top: 0;
        left: -150%;
        width: 60%;
        height: 100%;

        background: linear-gradient(120deg, transparent 0%, rgba(255, 255, 255, 0.05) 35%, rgba(255, 255, 255, 0.15) 50%, rgba(255, 255, 255, 0.05) 65%, transparent 100%);

        transform: skewX(-20deg);
        animation: shine 3s linear infinite;
    }
    .createThreadButton.disabled {
        pointer-events: none;
    }
    @keyframes shine {
        100% {
            left: 150%;
        }
    }
    .createThreadButton::before {
        font-family: 'Font Awesome 5 Pro';
        font-weight: 500;
        content: '\f055';
        font-size: 16px;
    }
    .crumbs > .crumb {
        white-space: wrap;
        font-size: 13px;
        color: #949494;
    }
    .crumbs > .crumb:last-child {
        font-weight: bold;
        font-size: 14px;
        color: #d6d6d6;
    }
    .crumbs > .sep {
        color: rgba(214, 214, 214, 0.35);
    }
    .selectedForumIcon {
        font-size: 24px;
        height: 44px;
        min-width: 44px;
        background-color: #192f27;
        border-radius: 12px;
        border: 2px solid #0b7d54;
    }
    .footerLeft {
        display: flex;
        gap: 15px;
    }
    .content {
        padding: 16px;
        background-color: var(--contentBackground);
        border-radius: 8px;
        min-height: 40vh;
    }
    .content.footerShown {
        border-radius: 8px 8px 0 0;
    }
    .searchContainer {
        margin-bottom: 20px;
        position: relative;
    }
    .searchContainer::before {
        color: #949494;
        font-family: 'Font Awesome 5 Pro';
        font-weight: 500;
        content: '\f002';
        float: left;
        margin-right: -36px;
        width: 36px;
        text-align: center;
        position: relative;
        line-height: 38px;
        font-size: 14px;
        pointer-events: none;
    }
    .searchContainer:focus-within::before {
        color: #d6d6d6;
    }
    .searchContainer input {
        width: 100%;
        box-sizing: border-box;
        outline: none;
        border-radius: 8px;
        border: none;
        padding-left: 36px;
        font-size: 14px;
        line-height: 38px;
        height: 38px;
        background-color: rgba(20, 20, 20, 0.5);
        color: #949494;
    }
    .searchContainer input:focus {
        box-shadow: 0 0 0px 2px rgb(46, 97, 74);
        color: #d6d6d6;
    }
    .searchContainer .clearBtn {
        position: absolute;
        right: 0;
        top: 0;
        width: 36px;
        height: 38px;
        border: none;
        background: transparent;
        cursor: pointer;
        font-family: 'Font Awesome 5 Pro';
        font-weight: 500;
        font-size: 16px;
        color: #949494;
    }

    .searchContainer .clearBtn::before {
        content: '\f00d'; /* fa-times */
    }

    .searchContainer .clearBtn:hover {
        color: #d6d6d6;
    }
    .forums {
        display: grid;
        gap: 16px;
    }
    .forumContainer > .title {
        font-size: 14px;
        padding-bottom: 8px;
        color: #949494;
        padding-left: 4px;
    }

    .forumItems {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: 6px 8px;
    }

    @media (max-width: 720px) {
        .forumItems {
            grid-template-columns: repeat(2, 1fr);
        }
    }

    @media (max-width: 520px) {
        .forumItems {
            grid-template-columns: repeat(1, 1fr);
        }
    }

    .forumInfo {
        display: flex;
        align-items: center;
        gap: 8px;
    }
    .forumItem {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 10px;
        padding: 8px 11px;
        border-radius: 8px;
        background: #ffffff08;
        border: 2px solid #24242400;
        max-width: 500px;
        cursor: pointer;
        transition: 0.2s ease;
    }
    .forumItem:hover {
        background: #ffffff12;
    }
    .forumItem .forumIcon {
        border-radius: 6px;
        color: #949494;
        font-size: 16px;
        transition: 0.2s ease;
    }
    .forumItem:hover .forumIcon,
    .forumItem:hover .forumIcon::before {
        color: #d6d6d6;
    }

    .hasChildren {
        width: 18px;
        height: 18px;
        border-radius: 6px;
        position: relative;
        flex: 0 0 auto;
        opacity: 0.9;
        transition: 0.2s ease;
    }
    .hasChildren::before {
        font-size: 15px;
        font-family: 'Font Awesome 5 Pro';
        font-weight: 400;
        content: '\f105';
        padding-left: 6px;
    }

    .forumItem.isSelected .hasChildren {
        transform: rotate(90deg) translateX(1px);
        color: #00ba78;
    }

    .forumItem.isActive {
        border-color: #0b7d54;
        background: #192f27;
    }
    .forumItem.isActive,
    .forumItem.isSelected {
        color: #00ba78;
    }
    .forumItem.isActive .forumIcon {
        color: #00ba78;
        border-color: transparent;
    }
    .forumItem.isActive .forumIcon::before {
        color: #00ba78;
    }
    .forumItem.isActive .hasChildren::before {
        border-left-color: #2b3a35;
    }
    .subBlock {
        grid-column: 1 / -1;
        overflow: hidden;
        padding: 10px 15px;
        border-radius: 8px;
        background: #ffffff08;
        background-size: cover;
    }
    .subTitle {
        font-size: 13px;
        color: #949494;
        margin-bottom: 4px;
        font-weight: 500;
    }
    .subForumItems {
        display: flex;
        align-items: flex-start;
        flex-wrap: wrap;
        gap: 6px;
    }
    .subForumItem {
        display: flex;
        align-items: start;
        justify-content: space-between;
        gap: 8px;

        padding: 8px 12px;
        border-radius: 8px;
        background: #ffffff08;
        border: 2px solid #24242400;
        cursor: pointer;
        line-height: 14px;
        color: #d6d6d6;

        transition: 0.2s ease;
    }

    .subForumItem .subForumName {
        display: flex;
        align-items: center;
        gap: 6px;
    }

    .subForumItem .subForumName > .forumIcon {
        font-size: 16px;
        color: #949494;
    }

    .subForumItem .subForumName > .name {
        font-weight: 600;
    }

    .subForumItem.noChildren {
        padding-right: 12px;
    }

    .subForumItem .hasChildren {
        margin-top: 1px;
        height: 12px;
        width: 12px;
    }

    .subForumItem .hasChildren::before {
        font-size: 12px;
    }

    .subForumItem:hover {
        border-color: rgb(233 233 233 / 0.15);
        background: #ffffff12;
    }

    .subForumItem.isActive {
        border-color: rgb(46, 97, 74);
        background: #2b3a35;
    }
    .subForumItem.isSelected,
    .subForumItem.isActive,
    .subForumItem.isSelected .forumIcon,
    .subForumName .forumIcon.isActive {
        color: #00ba78;
    }
    .subForumItem.isSelected .hasChildren {
        transform: rotate(90deg) translateX(-1px) translateY(-2px);
        color: #00ba78;
    }

    .subForumContainer {
        background: rgb(40, 40, 40);
        border-radius: 8px;
    }

    .subForumChildren {
        display: grid;
        overflow: hidden;
        z-index: 1;
    }

    .subForumChildren.hidden {
        visibility: hidden;
        pointer-events: none;
        opacity: 0;
        height: 0;
        z-index: -1;
    }

    .subForumChild {
        font-size: 13px;
        padding: 8px 15px;
        cursor: pointer;
        border-radius: 8px;
        border: 1px solid transparent;
    }
    .subForumChild:hover {
        background: rgba(255, 255, 255, 0.02);
    }
    .subForumChild.isActive {
        border-color: rgb(46, 97, 74);
        background: #2b3a35;
        color: #00ba78;
    }

    .contentContainer {
        padding: 0 4px 4px 4px;
    }
</style>
