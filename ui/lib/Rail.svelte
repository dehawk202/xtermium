<script lang="ts">
  // The collapsed sidebar's sessions: New session, then each top-level folder and each session
  // outside a folder as an icon, in the sidebar's order. A session's icon opens it; a folder's
  // opens a menu of the sessions in it, headed by subfolder, ending with New session here. So the
  // sidebar can stay collapsed and still be where sessions are. A folder wears the dot and the
  // amber edge of the open sessions inside it, so you can tell where a tab came from.
  import { _ } from "./i18n.svelte";
  import type { Session } from "./api";
  import Icon from "./Icon.svelte";
  import { menu } from "./menu";
  import SessionIcon from "./SessionIcon.svelte";
  import { app, folderColor, lookOf, sessionKey } from "./state.svelte";

  let {
    onopen,
    onnew,
    onimport,
    live = new Map(),
    current = "",
  }: {
    onopen: (s: Session) => void;
    /** A new session, in this folder ("" for none). */
    onnew: (folder: string) => void;
    /** Opens Import sessions, offered while nothing is saved. */
    onimport: () => void;
    /** Sessions with a tab, by `sessionKey`: true while one is connected (Sidebar.svelte's). */
    live?: Map<string, boolean>;
    /** The `sessionKey` of the focused tab's session. */
    current?: string;
  } = $props();

  const nameOf = (p: string) => p.split("/").pop() ?? p;
  const parentOf = (p: string) => p.split("/").slice(0, -1).join("/");
  const folderOf = (p: string) => app.store.folder.find((f) => f.path === p);
  /** Every folder there is, stored or there through a session, as the sidebar's tree has them. */
  const folders = $derived(
    new Set([...app.store.folder.map((f) => f.path), ...app.store.session.flatMap((s) => s.folder.split("/").filter(Boolean).map((_, i, p) => p.slice(0, i + 1).join("/")))]),
  );
  /** Sibling folders in the sidebar's order: the ones dragged into place first, then by name. */
  const rank = (p: string) => folderOf(p)?.order || Infinity;
  const childrenOf = (p: string) => [...folders].filter((f) => parentOf(f) === p).sort((a, b) => rank(a) - rank(b) || nameOf(a).localeCompare(nameOf(b)));
  const top = $derived(childrenOf(""));
  const loose = $derived(app.store.session.filter((s) => !s.folder));
  /** Nothing saved: the rail offers New session as the primary, and Import. */
  const empty = $derived(!top.length && !loose.length);
  const within = (s: Session, p: string) => s.folder === p || s.folder.startsWith(p + "/");
  const countIn = (p: string) => app.store.session.filter((s) => within(s, p)).length;
  /** A folder's state, as its open sessions give it: true when one is connected, false when all
   *  of them are disconnected, undefined with none open. So a collapsed folder still shows the dot. */
  const liveIn = (p: string) => {
    const on = app.store.session.filter((s) => within(s, p)).map((s) => live.get(sessionKey(s))).filter((v) => v !== undefined);
    return on.length ? on.includes(true) : undefined;
  };
  const currentIn = (p: string) => app.store.session.some((s) => within(s, p) && sessionKey(s) === current);

  /** A folder's sessions as its menu lists them: its own first, then each subfolder's, depth first. */
  function groups(p: string): { path: string; sessions: Session[] }[] {
    return [{ path: p, sessions: app.store.session.filter((s) => s.folder === p) }, ...childrenOf(p).flatMap(groups)].filter((g) => g.sessions.length);
  }

  /** The folder whose menu is open, and where: beside the rail, level with the icon. */
  let shown = $state<{ path: string; x: number; y: number } | null>(null);
  function show(e: MouseEvent, path: string) {
    e.stopPropagation();
    if (shown?.path === path) return void (shown = null);
    const r = (e.currentTarget as HTMLElement).getBoundingClientRect();
    // Right to left the rail is at the window's right; a menu there opens leftwards from x (app.css).
    shown = { path, x: document.documentElement.dir === "rtl" ? r.left - 5 : r.right + 5, y: r.top };
  }
  // The menu goes when its folder does (a sync, a delete elsewhere).
  $effect(() => {
    if (shown && !folders.has(shown.path)) shown = null;
  });

  /** A folder's icon: its own, or a closed folder, in its mark's colour as the sidebar draws it. */
  const folderLook = (p: string) => {
    const tint = folderColor(p);
    return { icon: folderOf(p)?.icon || "folder", color: tint && `oklab(from ${tint} clamp(var(--tint-lo), l, var(--tint-hi)) a b)`, still: !!folderOf(p)?.still };
  };
</script>

<svelte:window onclick={() => (shown = null)} />

{#snippet session(s: Session)}
  {@const look = lookOf(s)}
  {@const on = live.get(sessionKey(s))}
  <button class="flat icon-only slot" class:live={on === true} class:idle={on === false} class:current={current === sessionKey(s)} title={s.name} aria-label={s.name} onclick={() => onopen(s)}>
    <SessionIcon icon={look.icon} color={look.color} still={look.still} size={16} />
  </button>
{/snippet}

<!-- With nothing saved, the rail says what the empty sidebar does: New session is the primary, and
     Import sits under it. -->
<button class="icon-only" class:flat={!empty} class:primary={empty} title={_("New session")} aria-label={_("New session")} onclick={() => onnew("")}><Icon name="plus" size={16} /></button>
{#if empty}
  <button class="flat icon-only" title={_("Import sessions")} aria-label={_("Import sessions")} onclick={onimport}><Icon name="import" size={16} /></button>
{/if}
{#if !empty}
  <div class="items" role="list" aria-label={_("Sessions")}>
    {#each top as p (p)}
      {@const look = folderLook(p)}
      {@const label = _("{folder}: {n} sessions", { folder: nameOf(p), n: countIn(p) })}
      {@const on = liveIn(p)}
      <button class="flat icon-only slot" class:on={shown?.path === p} class:live={on === true} class:idle={on === false} class:current={currentIn(p)} title={label} aria-label={label} aria-haspopup="menu" aria-expanded={shown?.path === p} onclick={(e) => show(e, p)}>
        <SessionIcon icon={look.icon} color={look.color} still={look.still} size={16} />
      </button>
    {/each}
    {#each loose as s (sessionKey(s))}{@render session(s)}{/each}
  </div>
{:else}
  <!-- the rest of the rail stays empty, so the foot keeps to the bottom -->
  <span class="grow"></span>
{/if}

{#if shown}
  {@const p = shown.path}
  <div class="menu rail-menu" aria-label={nameOf(p)} {@attach menu(() => (shown = null), shown.x, shown.y)}>
    <span class="head">{nameOf(p)}</span>
    {#each groups(p) as g (g.path)}
      {#if g.path !== p}<span class="group sep">{g.path.slice(p.length + 1)}</span>{/if}
      {#each g.sessions as s (sessionKey(s))}
        {@const look = lookOf(s)}
        {@const on = live.get(sessionKey(s))}
        <!-- act first, then close: `p` and `s` are read from `shown`, which closing clears -->
        <button class:live={on === true} class:idle={on === false} class:current={current === sessionKey(s)} onclick={() => (onopen(s), (shown = null))}>
          <SessionIcon icon={look.icon} color={look.color} still={look.still} size={16} /><span class="name">{s.name}</span>
        </button>
      {/each}
    {/each}
    <button class="sep" onclick={() => (onnew(p), (shown = null))}><Icon name="plus" size={16} />{_("New session here")}</button>
  </div>
{/if}

<style>
  /* the sessions between New session and the foot: as many as fit, then they scroll */
  .items {
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
    width: 100%;
    /* the hairline right under New session's square, level with the tab strip's line beside it */
    padding-block-start: 5px;
    border-top: 1px solid var(--line);
    overflow-y: auto;
    scrollbar-width: none;
  }
  .grow {
    flex: 1;
  }
  .slot {
    flex: none;
    position: relative;
  }
  /* the open folder's icon, as a pressed tool */
  .slot.on {
    background: var(--raised);
  }
  /* the focused tab's session: lifted, with the focus ring's amber at its edge, as in the sidebar */
  .slot.current {
    background: var(--raised);
    box-shadow: inset 2px 0 0 var(--gold);
  }
  :global([dir="rtl"]) .slot.current {
    box-shadow: inset -2px 0 0 var(--gold);
  }
  /* a session with a tab open: the sidebar's dot on its icon's corner, hollow once disconnected */
  :is(.slot, .rail-menu button):is(.live, .idle)::after {
    content: "";
    position: absolute;
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: var(--ok);
    box-shadow: 0 0 0 1.5px rgb(var(--panel-rgb));
  }
  .slot:is(.live, .idle)::after {
    inset-inline-end: 3px;
    bottom: 3px;
  }
  .rail-menu button:is(.live, .idle) {
    position: relative;
  }
  .rail-menu button:is(.live, .idle)::after {
    inset-inline-start: calc(8px + 16px - 5px);
    top: 50%;
  }
  :is(.slot, .rail-menu button).idle::after {
    background: rgb(var(--panel-rgb));
    border: 1.5px solid var(--weak);
  }
  .rail-menu {
    max-height: calc(100vh - var(--titlebar) - 2 * 13px);
    overflow-y: auto;
  }
  .rail-menu button {
    gap: var(--gap);
  }
  .rail-menu button.current {
    background: var(--raised);
  }
  .name {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  /* the folder's name, then a subfolder's, over the rows they hold */
  .head,
  .group {
    padding: 3px 8px;
    color: var(--weak);
    font-size: var(--t-s);
  }
  .head {
    color: var(--fg);
    font-weight: 600;
  }
</style>
