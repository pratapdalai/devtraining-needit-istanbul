/* =====================================================================
   Auto KCI – record pages: order, case / service request, incident
   Layout (from the approved mock-up): header with stage bar and actions -> AI summary -> activity timeline
   -> related tabs (tasks, SLA, ...) | right rail: customer & details (editable), SLA, Copilot.
   Uses the shared helpers from workspace.js (window.WS). All data is escaped with esc().
   ===================================================================== */
(() => {
  'use strict';
  const W = window.WS;
  const { $, esc, api, toast, avatar, ago, day, money, SPARK, ICON, eventName, statePill, stagePill, priPill, slaCell } = W;

  // ---------------- icons ----------------
  const I = {
    lock: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="5" y="11" width="14" height="10" rx="2"/><path d="M8 11V7a4 4 0 018 0v4"/></svg>',
    chat: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 12a8 8 0 01-11.6 7.1L4 20l1-4.6A8 8 0 1121 12z"/></svg>',
    mail: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="5" width="18" height="14" rx="2"/><path d="M3 7l9 6 9-6"/></svg>',
    tick: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><path d="M5 12.5l4.5 4.5L19 7.5"/></svg>',
    warn: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><circle cx="12" cy="12" r="9"/><path d="M12 7v6M12 16.5v.5"/></svg>',
    pin: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21s-7-6.2-7-11a7 7 0 1114 0c0 4.8-7 11-7 11z"/><circle cx="12" cy="10" r="2.5"/></svg>',
    cal: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><rect x="3" y="5" width="18" height="16" rx="2"/><path d="M3 10h18M8 3v4M16 3v4"/></svg>',
    pound: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M17 20H6c2-1.5 3-3.5 3-6V8a4 4 0 017.5-2M6 13h8"/></svg>',
    tag: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 12l-8 8-9-9V3h8z"/><circle cx="7.5" cy="7.5" r="1.5"/></svg>',
    team: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="9" cy="8" r="3.5"/><path d="M2 20c0-3.5 3-6 7-6s7 2.5 7 6M16 4a3.5 3.5 0 010 7M22 20c0-3-2-5.2-5-5.8"/></svg>',
    dots: '<svg viewBox="0 0 24 24" fill="currentColor"><circle cx="5" cy="12" r="2"/><circle cx="12" cy="12" r="2"/><circle cx="19" cy="12" r="2"/></svg>',
    send: '<svg class="i" viewBox="0 0 24 24" fill="currentColor"><path d="M3.4 20.4l17.4-7.5c.8-.4.8-1.5 0-1.8L3.4 3.6c-.7-.3-1.4.2-1.4.9L2 9.1c0 .5.4.9.9 1l12.1 1.9-12.1 1.9c-.5.1-.9.5-.9 1l0 4.6c0 .7.7 1.2 1.4.9z"/></svg>',
    arrow: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M5 12h14M13 6l6 6-6 6"/></svg>',
  };
  const time = (ts) => new Date(ts).toLocaleTimeString('en-GB', { hour: 'numeric', minute: '2-digit', hour12: true });
  const when = (ts) => ts ? `${day(ts)}, ${time(ts)}` : '';
  const first = (name) => String(name || '').trim().split(/\s+/)[0] || 'the customer';
  const isoDate = (d) => `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;
  const nextWorkingDay = () => { const d = new Date(); do d.setDate(d.getDate() + 1); while (d.getDay() === 0); return d; };

  // ---------------- state (kept per record while the agent moves around) ----------------
  const R = { type: null, id: null, data: null };
  const mem = {};                                   // per record: tab, filter, shown, summary, chat, composer mode
  const M = () => (mem[`${R.type}:${R.id}`] ||= { tab: null, filter: 'all', shown: 25, summary: null, showSummary: false, chat: [], mode: 'public' });
  const URLS = { order: (id) => `/api/ws/orders/${id}`, case: (id) => `/api/ws/cases/${id}`, incident: (id) => `/api/ws/incidents/${id}` };

  // =====================================================================
  // ENTRY POINTS (called by workspace.js)
  // =====================================================================
  W.renderRecord = async (page, type, id) => {
    R.data = null;
    page.innerHTML = `<section class="view"><div class="card" style="padding:22px;display:grid;gap:12px"><div class="skel" style="height:14px;width:20%"></div><div class="skel" style="height:26px;width:55%"></div><div class="skel" style="height:56px"></div></div>
      <div class="rec-body"><div class="card skel" style="height:360px"></div><div class="card skel" style="height:360px"></div></div></section>`;
    const data = await api(URLS[type](id));
    if (location.hash !== `#/${type}/${id}`) return;          // the agent already moved on: don't draw over the new page
    R.type = type; R.id = id; R.data = data;
    draw(page);
  };
  // quiet refresh (every 15 s) only when the agent is not in the middle of something
  W.refreshRecord = async () => {
    if (!R.data) return;
    const busy = document.activeElement?.closest?.('#page') && ['INPUT', 'TEXTAREA', 'SELECT'].includes(document.activeElement.tagName);
    if (busy || $('compText')?.value.trim() || document.querySelector('.inl.dirty') || document.querySelector('.menu.open')) return;
    const at = location.hash;
    const fresh = await api(URLS[R.type](R.id));
    if (location.hash !== at || JSON.stringify(fresh) === JSON.stringify(R.data)) return;
    R.data = fresh; draw($('page'), true);
  };
  async function reload() {
    const at = location.hash;
    const data = await api(URLS[R.type](R.id));
    if (location.hash !== at) return;
    R.data = data;
    draw($('page'), true);
    W.loadNav();
  }

  // =====================================================================
  // PAGE
  // =====================================================================
  function draw(page, keepScroll = false) {
    const y = window.scrollY;
    const vm = model();
    const m = M();
    if (!m.tab || !vm.tabs.some((t) => t.key === m.tab)) m.tab = vm.tabs[0]?.key;
    page.innerHTML = `<section class="view">
      <div class="crumbs"><a href="#/home">Home</a><span>/</span><a href="${vm.back.href}">${esc(vm.back.label)}</a><span>/</span><span class="mono">${esc(vm.ref)}</span></div>
      <section class="card rec-h">
        <div class="rec-top">
          <div style="min-width:0;flex:1">
            <div class="id"><span class="mono num" style="font-size:14px">${esc(vm.ref)}</span>${vm.pills}</div>
            <h1>${esc(vm.title)}</h1>
            <div class="sub">${vm.sub.map(([icon, text]) => `<span>${I[icon] || ''}${esc(text)}</span>`).join('')}</div>
          </div>
          <div class="rec-acts">
            <div class="acts">
              <button class="btn ai" type="button" id="sumBtn">${SPARK}AI summary</button>
              ${vm.canEmail ? `<button class="btn" type="button" id="mailBtn">${I.mail}Draft email</button>` : ''}
              ${vm.canSave ? '<button class="btn" type="button" id="saveBtn">Save</button>' : ''}
              ${vm.more.length ? `<div class="menu" id="moreMenu"><button class="btn" type="button" aria-label="More actions">${I.dots}More</button>
                <div class="menu-list">${vm.more.map((x, i) => `<button type="button" class="${x.danger ? 'danger' : ''}" data-more="${i}">${esc(x.label)}</button>`).join('')}</div></div>` : ''}
              ${vm.advance ? `<button class="btn primary" type="button" id="advBtn" ${vm.advance.enabled ? '' : 'disabled'} title="${esc(vm.advance.block || '')}">${esc(vm.advance.label)}${I.arrow}</button>` : ''}
            </div>
            ${vm.advance && !vm.advance.enabled && vm.advance.block ? `<div class="lock">${I.lock}${esc(vm.advance.block)}</div>` : ''}
          </div>
        </div>
        ${vm.banner ? `<div class="banner ${vm.banner.cls}">${I.warn}<span>${esc(vm.banner.text)}</span></div>` : ''}
        <div id="sumPanel"></div>
        <div class="stages" style="grid-template-columns:repeat(${vm.stages.length},1fr)">${vm.stages.map((s, i) => `
          <div class="stg ${s.cls}"><span class="dot">${s.cls === 'done' ? '✓' : i + 1}</span><div style="min-width:0"><b>${esc(s.name)}</b><span>${esc(s.sub || '')}</span>
            ${s.mini ? `<div class="mini">${s.mini.map((x) => `<i class="${x}"></i>`).join('')}</div>` : ''}</div></div>`).join('')}</div>
      </section>

      <div class="rec-body">
        <div class="col">
          <section class="card">
            <div class="card-h"><h3>Activity</h3>
              <div class="seg" id="actFilter">${[['all', 'All'], ['ai', 'AI updates'], ['customer', 'Customer'], ['internal', 'Internal']].map(([k, l]) => `<button type="button" data-f="${k}" class="${m.filter === k ? 'on' : ''}">${l}</button>`).join('')}</div></div>
            ${composer(vm)}
            <ul class="tl" id="timeline">${timeline(vm)}</ul>
            <div class="more-row" id="moreRow"></div>
          </section>
          <section class="card">
            <div class="rtabs" id="rtabs">${vm.tabs.map((t) => `<button type="button" class="t ${t.key === m.tab ? 'on' : ''}" data-t="${t.key}">${esc(t.label)}<em>${t.count}</em></button>`).join('')}<span class="tab-act" id="tabAct"></span></div>
            <div id="rpane"></div>
          </section>
        </div>
        <aside class="rail">${vm.rail}
          <section class="card copilot">
            <div class="card-h"><h3>${SPARK}Copilot</h3><span class="pill ai">Knows this ${esc(vm.noun)}</span></div>
            <div class="cp-body" id="cpBody"></div>
            <div class="cp-chips">${vm.chips.map((c) => `<button type="button">${esc(c)}</button>`).join('')}</div>
            <div class="cp-in"><input id="cpIn" placeholder="Ask about this ${esc(vm.noun)}…" autocomplete="off"><button type="button" id="cpSend" aria-label="Send">${I.send}</button></div>
          </section>
        </aside>
      </div></section>`;

    wire(vm);
    drawTab(vm); applyFilter(); drawSummary(); drawChat();
    if (keepScroll) window.scrollTo({ top: y });
  }

  // =====================================================================
  // VIEW MODEL per record type
  // =====================================================================
  function model() {
    return R.type === 'order' ? orderModel(R.data) : R.type === 'case' ? caseModel(R.data) : incidentModel(R.data);
  }

  const slaPill = (s) => !s ? '' : s.breached ? '<span class="pill bad">SLA breached</span>'
    : s.state !== 'In Progress' ? '<span class="pill ok">SLA met</span>'
    : s.health === 'at_risk' ? '<span class="pill warn">SLA at risk</span>' : '<span class="pill ok">SLA on track</span>';

  function ring(s, title) {
    if (!s) return `<div class="ring"><div><b>${esc(title)}</b><span>No SLA</span></div></div>`;
    const pct = Math.min(100, s.state === 'In Progress' ? s.pct_used : 100);
    const colour = { breached: '#C2352B', at_risk: '#E39200' }[s.health] || '#12804A';
    const label = s.breached ? 'Breached' : s.state === 'Completed' ? 'Met' : s.state === 'Cancelled' ? 'Cancelled' : s.health === 'at_risk' ? 'At risk' : 'On track';
    return `<div class="ring"><svg viewBox="0 0 42 42"><circle cx="21" cy="21" r="16" fill="none" stroke="#EEF1F6" stroke-width="4.5"/>
      <circle cx="21" cy="21" r="16" fill="none" stroke="${colour}" stroke-width="4.5" pathLength="100" stroke-dasharray="${pct} ${100 - pct}" stroke-linecap="round" transform="rotate(-90 21 21)"/>
      <text x="21" y="24" text-anchor="middle" font-family="Bricolage Grotesque Variable, sans-serif" font-size="9" font-weight="700" fill="#0B1736">${s.state === 'In Progress' ? s.pct_used + '%' : '✓'}</text></svg>
      <div><b>${esc(title)}: ${label}</b><span>${esc(s.name)}<br>${esc(s.time_left || (s.took ? `took ${s.took}` : ''))}</span></div></div>`;
  }
  const slaList = (slas) => slas.length ? `<ul class="slas">${slas.map(({ label, sla: s }) => `<li><div><span>${esc(label)}</span><span>${esc(s?.time_left || (s?.state === 'Completed' ? (s.breached ? 'Breached' : 'Met') : s ? s.state : 'Not started'))}</span></div>
    <div class="slabar"><i class="g-${esc(s?.health || 'on_track')}" style="width:${s ? (s.state === 'In Progress' ? Math.min(100, s.pct_used) : 100) : 0}%"></i></div></li>`).join('')}</ul>` : '';

  const kvInput = (field, value, type = 'input') => type === 'textarea'
    ? `<textarea class="inl" data-f="${field}" rows="2">${esc(value || '')}</textarea>`
    : `<input class="inl" data-f="${field}" value="${esc(value || '')}">`;
  const kvSelect = (field, value, options) => `<select class="inl" data-f="${field}">${options.map((o) => `<option ${o === value ? 'selected' : ''}>${esc(o)}</option>`).join('')}</select>`;

  // ---------------- ORDER ----------------
  function orderModel(o) {
    const ended = ['Completed', 'Closed', 'Cancelled'].includes(o.status);
    const idx = o.stages.indexOf(o.current_stage);
    const hist = (stage) => o.history.find((h) => h.to_stage === stage)?.changed_at;
    const stages = o.stages.map((name, i) => {
      const tasks = o.tasks.filter((t) => t.stage === name);
      const closed = tasks.filter((t) => ['Closed', 'Cancelled'].includes(t.state)).length;
      const done = i < idx || ['Completed', 'Closed'].includes(o.status);
      if (done) return { name, cls: 'done', sub: name === 'Validation' ? `${o.validation?.verdict === 'PASS' ? 'AI validated' : 'Approved'} · ${when(o.tasks.find((t) => t.task_type === 'VALIDATION')?.closed_at || hist(name))}` : when(hist(name)) };
      if (i === idx && o.status !== 'Cancelled') return {
        name, cls: o.status === 'On Hold' ? 'hold' : 'now',
        sub: o.status === 'On Hold' ? 'On hold: incident open' : tasks.length ? `${closed} of ${tasks.length} task${tasks.length > 1 ? 's' : ''} closed` : 'In progress',
        mini: tasks.length > 1 ? tasks.map((t) => (['Closed', 'Cancelled'].includes(t.state) ? 'd' : ['Open', 'In Progress'].includes(t.state) ? 'n' : '')) : null,
      };
      return { name, cls: '', sub: name === 'Completed' ? 'Final update' : tasks.length ? tasks.map((t) => t.name).join(', ') : '' };
    });
    const taskSlas = o.tasks.filter((t) => t.task_type !== 'VALIDATION').map((t) => ({ label: t.name, sla: t.sla }));
    const more = [];
    if (['In Progress', 'On Hold'].includes(o.status)) {
      more.push({ label: 'Raise incident', run: raiseIncident });
      more.push({ label: 'New case for this order', run: () => W.openNewCase({ order_number: o.order_number }) });
    }
    if (o.status === 'Completed') more.push({ label: 'Close order', run: () => act(`/api/ws/orders/${o.id}/close`, {}, 'Order closed') });
    more.push({ label: 'Open customer tracking page', run: () => window.open(`/track.html?id=${o.id}`, '_blank', 'noopener') });
    if (['In Progress', 'On Hold'].includes(o.status)) more.push({ label: 'Cancel order', danger: true, run: cancelOrder });

    return {
      noun: 'order', ref: o.order_number, title: `${o.customer_name} · ${o.product_name}`,
      back: { href: `#/list/orders/${{ 'In Progress': 'open', 'On Hold': 'open', Completed: 'completed', Closed: 'closed', Cancelled: 'cancelled' }[o.status]}`, label: 'Orders' },
      pills: `${stagePill(o.current_stage, o.status)}${o.status !== 'In Progress' && o.status !== 'On Hold' ? statePill(o.status) : ''}${slaPill(o.order_sla)}${o.needs_attention ? '<span class="pill warn">Needs attention</span>' : ''}`,
      sub: [['pin', `${o.address || ''}${o.pincode ? ' · ' + o.pincode : ''}`], ['cal', `Placed ${when(o.created_at)}`],
        ...(o.eta ? [['tag', `Expected ready ${o.eta}`]] : []), ['pound', `${money(o.monthly_total_gbp)} per month`]],
      canEmail: true, canSave: !['Closed', 'Cancelled'].includes(o.status), more,
      advance: ended ? null : { label: `Move to ${o.next_stage}`, enabled: o.can_advance, run: advanceOrder,
        block: o.advance_block && `${o.advance_block}. The AI Stage Agent then moves the order on by itself.` },
      banner: o.status === 'Cancelled' ? { cls: 'grey', text: `Cancelled ${when(o.cancelled_at)}: ${o.cancel_reason || ''}` }
        : o.status === 'Closed' ? { cls: 'grey', text: `Closed ${when(o.closed_at)}.` }
        : stageHold(o) ? { cls: 'bad', text: `AI Stage Agent held this order: ${stageHold(o)}` }
        : o.validation?.verdict === 'REVIEW' && o.tasks[0]?.state !== 'Closed' ? { cls: 'bad', text: `AI Order Agent asked for a review: ${o.validation.issues.join('; ')}` } : null,
      stages,
      kcis: o.kcis, activities: o.activities,
      customer: { name: o.customer_name, email: o.customer_email },
      chips: ["Why hasn't this moved on?", `Draft a reply to ${first(o.customer_name)}`, "What's the SLA risk?", 'Summarise the internal notes'],
      tabs: [
        { key: 'tasks', label: 'Order tasks', count: o.tasks.length, draw: orderTasksTab },
        { key: 'sla', label: 'Order SLA', count: o.slas.length, draw: () => slaTab(o.slas) },
        { key: 'decisions', label: 'AI decisions', count: o.decisions.length, draw: decisionsTab },
        { key: 'inc', label: 'Incidents', count: o.incidents.length, draw: orderIncidentsTab },
        { key: 'cases', label: 'Cases', count: o.cases.length, draw: orderCasesTab },
      ],
      rail: `<section class="card pad">
          <div class="person">${avatar(o.customer_name, 44)}<div><b>${esc(o.customer_name)}</b><span>Customer since ${esc(day(o.created_at))}</span></div></div>
          <dl class="kv">
            <dt>Email</dt><dd>${kvInput('customer_email', o.customer_email)}</dd>
            <dt>Mobile</dt><dd>${kvInput('customer_phone', o.customer_phone)}</dd>
            <dt>Address</dt><dd>${kvInput('address', o.address, 'textarea')}</dd>
            <dt>Postcode</dt><dd>${esc(o.pincode || '—')}</dd>
            <dt>Plan</dt><dd>${esc(o.product_name)}</dd>
            <dt>Add-ons</dt><dd>${esc((o.addons || []).map((a) => a.name).join(', ') || 'None')}</dd>
            <dt>Monthly</dt><dd>${money(o.monthly_total_gbp)}</dd>
            ${o.visit_slot ? `<dt>Engineer</dt><dd>${esc(o.visit_slot)}</dd>` : ''}
          </dl>
          ${o.customer_needs ? `<div class="needs"><span class="ai-tag">${SPARK}Told the Plan Advisor</span>${esc(o.customer_needs)}</div>` : ''}
        </section>
        <section class="card pad">${ring(o.order_sla, 'Order SLA')}${slaList(taskSlas)}</section>`,
    };
  }

  // ---------------- CASE / SERVICE REQUEST ----------------
  let CASE_OPTS = null;
  function caseModel(c) {
    const sr = c.type === 'Service Request';
    const noun = sr ? 'request' : 'case';
    const ended = ['Closed', 'Cancelled'].includes(c.state);
    const idx = c.stages.indexOf(c.state);
    const dates = { New: c.created_at, Resolved: c.resolved_at, Closed: c.closed_at };
    const stages = c.stages.map((name, i) => {
      if (c.state === 'Cancelled') return { name, cls: '', sub: '' };
      if (i < idx || c.state === 'Closed') return { name, cls: 'done', sub: when(dates[name]) || 'Done' };
      if (i === idx) {
        const open = c.tasks.filter((t) => ['Open', 'In Progress'].includes(t.state)).length;
        return { name, cls: 'now', sub: name === 'Open' && c.tasks.length ? `${c.tasks.length - open} of ${c.tasks.length} tasks closed` : name === 'New' ? `Opened ${ago(c.created_at)}` : when(dates[name]) };
      }
      return { name, cls: '', sub: name === 'Resolved' ? 'Customer told' : '' };
    });
    const opts = CASE_OPTS || { categories: { [c.type]: [c.category] }, priorities: [c.priority], teams: [c.assignment_group] };
    const more = [];
    if (!ended && c.state !== 'Resolved') more.push({ label: 'Add task', run: addCaseTask });
    if (c.order) more.push({ label: `Open order ${c.order.order_number}`, run: () => W.go(W.recordHash('order', c.order.id)) });
    if (['New', 'Open'].includes(c.state)) more.push({ label: `Cancel ${noun}`, danger: true, run: cancelCase });
    const advLabel = { Open: 'Start work', Resolved: 'Resolve', Closed: 'Close' }[c.next_stage];

    return {
      noun, ref: c.case_number, title: c.short_description,
      back: { href: `#/list/${sr ? 'requests' : 'cases'}/${c.state === 'Cancelled' && sr ? 'closed' : c.state.toLowerCase()}`, label: sr ? 'Service requests' : 'Cases' },
      pills: `<span class="pill ${sr ? 'ai' : 'blue'}">${esc(c.type)}</span>${statePill(c.state)}${priPill(c.priority)}${slaPill(c.case_sla)}`,
      sub: [['tag', c.category], ['team', c.assignment_group + (c.assigned_to ? ` · ${c.assigned_to}` : '')], ['cal', `Opened ${when(c.created_at)}`],
        ['chat', c.origin === 'Order Assistant' ? 'Opened by the Order Assistant' : 'Opened by an agent']],
      canEmail: true, canSave: !ended, more,
      advance: ended ? null : { label: advLabel, enabled: c.can_advance, block: c.advance_block, run: advanceCase },
      banner: c.state === 'Cancelled' ? { cls: 'grey', text: `Cancelled ${when(c.cancelled_at)}: ${c.cancel_reason || ''}` }
        : c.resolution ? { cls: 'ok', text: `Resolution: ${c.resolution}` } : null,
      stages, kcis: c.kcis, activities: c.activities,
      customer: { name: c.customer_name, email: c.customer_email },
      chips: ['What does the customer want?', 'What should I do next?', `Draft a reply to ${first(c.customer_name)}`],
      tabs: [
        { key: 'tasks', label: sr ? 'Request tasks' : 'Case tasks', count: c.tasks.length, draw: caseTasksTab },
        { key: 'sla', label: sr ? 'Request SLA' : 'Case SLA', count: c.slas.length, draw: () => slaTab(c.slas) },
      ],
      rail: `<section class="card pad">
          <div class="person">${avatar(c.customer_name, 44)}<div><b>${esc(c.customer_name)}</b><span>${esc(c.customer_email || '')}</span></div></div>
          <dl class="kv">
            <dt>Mobile</dt><dd>${esc(c.customer_phone || '—')}</dd>
            <dt>Order</dt><dd>${c.order ? `<a href="${W.recordHash('order', c.order.id)}" class="mono">${esc(c.order.order_number)}</a> · ${esc(c.order.product_name)}<span class="sub2">${esc(c.order.current_stage)} (${esc(c.order.status)})</span>` : 'No order'}</dd>
            <dt>Title</dt><dd>${kvInput('short_description', c.short_description, 'textarea')}</dd>
            <dt>Category</dt><dd>${kvSelect('category', c.category, opts.categories[c.type] || [c.category])}</dd>
            <dt>Priority</dt><dd>${kvSelect('priority', c.priority, opts.priorities)}</dd>
            <dt>Team</dt><dd>${kvSelect('assignment_group', c.assignment_group, opts.teams)}</dd>
            <dt>Assigned to</dt><dd>${kvInput('assigned_to', c.assigned_to)}</dd>
            <dt>Details</dt><dd>${kvInput('description', c.description, 'textarea')}</dd>
          </dl>
          ${c.triage ? `<div class="needs"><span class="ai-tag">${SPARK}${c.triage.ai_source === 'openai' ? 'Triaged by AI' : 'Triaged (rules, AI demo mode)'}</span>${esc(c.triage.summary || '')}</div>` : ''}
        </section>
        <section class="card pad">${ring(c.case_sla, sr ? 'Request SLA' : 'Case SLA')}${slaList(c.tasks.map((t) => ({ label: t.task_number, sla: t.sla })))}</section>`,
    };
  }

  // ---------------- INCIDENT ----------------
  function incidentModel(i) {
    const ended = i.state === 'Closed';
    const idx = i.stages.indexOf(i.state);
    const dates = { New: i.created_at, Resolved: i.resolved_at, Closed: i.closed_at };
    const stages = i.stages.map((name, n) => n < idx || ended ? { name, cls: 'done', sub: when(dates[name]) || 'Done' }
      : n === idx ? { name, cls: 'now', sub: name === 'Open' && i.tasks.length ? `${i.tasks.filter((t) => t.state === 'Done').length} of ${i.tasks.length} tasks done` : when(dates[name]) || 'Now' }
      : { name, cls: '', sub: name === 'Resolved' ? 'Customers told' : '' });
    const more = [];
    if (['New', 'Open'].includes(i.state)) more.push({ label: 'Add task', run: addIncidentTask });
    if (i.order_id) more.push({ label: `Open order ${i.order_number}`, run: () => W.go(W.recordHash('order', i.order_id)) });
    if (i.parent_id) more.push({ label: `Open major incident ${i.parent_number}`, run: () => W.go(W.recordHash('incident', i.parent_id)) });
    const advLabel = { Open: 'Start work', Resolved: 'Resolve', Closed: 'Close' }[i.next_stage];
    const tabs = [
      { key: 'tasks', label: 'Incident tasks', count: i.tasks.length, draw: incidentTasksTab },
      { key: 'sla', label: 'Incident SLA', count: i.slas.length, draw: () => slaTab(i.slas) },
    ];
    if (i.is_major) tabs.splice(1, 0, { key: 'affected', label: 'Affected customers', count: i.children.length, draw: affectedTab });

    return {
      noun: 'incident', ref: i.incident_number, title: i.description,
      back: { href: `#/list/incidents/${i.state === 'New' ? 'new' : i.state === 'Open' ? 'open' : 'closed'}`, label: 'Incidents' },
      pills: `${i.is_major ? '<span class="tag">MAJOR</span>' : ''}${statePill(i.state)}${priPill(i.priority)}${slaPill(i.incident_sla)}`,
      sub: i.is_major ? [['pin', `Postcode ${i.pincode}`], ['team', i.assignment_group], ['cal', `Raised ${when(i.created_at)}`], ['tag', `${i.children.length} customers affected`]]
        : [['team', i.assignment_group || ''], ['cal', `Raised ${when(i.created_at)}`], ['tag', `${i.customer_name || ''} · ${i.order_number || ''}`]],
      canEmail: !!i.order_id, canSave: false, more,
      advance: ended ? null : { label: advLabel, enabled: i.can_advance, block: i.advance_block, run: advanceIncident },
      banner: i.resolution ? { cls: 'ok', text: `Resolution: ${i.resolution}` } : i.parent_id ? { cls: 'grey', text: `Part of major incident ${i.parent_number}: tasks and resolution are managed there.` } : null,
      stages, kcis: i.kcis.filter((k) => k.incident_id === i.id), activities: i.activities,
      customer: i.order_id ? { name: i.customer_name } : null,
      chips: i.is_major ? ['Who is affected?', "What's left to fix?", 'Draft a status update for customers'] : ["What's left to fix?", 'How is the customer affected?', `Draft a reply to ${first(i.customer_name)}`],
      tabs,
      rail: `<section class="card pad">
          ${i.is_major ? `<div class="person"><span class="av c1" style="width:44px;height:44px">${I.pin}</span><div><b>Postcode ${esc(i.pincode)}</b><span>${i.children.length} customers affected</span></div></div>`
            : `<div class="person">${avatar(i.customer_name, 44)}<div><b>${esc(i.customer_name || '—')}</b><span><a href="${W.recordHash('order', i.order_id)}" class="mono">${esc(i.order_number || '')}</a></span></div></div>`}
          <dl class="kv"><dt>Priority</dt><dd>${esc(i.priority)}</dd><dt>Team</dt><dd>${esc(i.assignment_group || '—')}</dd>
            <dt>Stage hit</dt><dd>${esc(i.stage || '—')}</dd><dt>Raised</dt><dd>${esc(when(i.created_at))}</dd>
            ${i.parent_number ? `<dt>Major</dt><dd><a href="${W.recordHash('incident', i.parent_id)}" class="mono">${esc(i.parent_number)}</a></dd>` : ''}</dl>
        </section>
        <section class="card pad">${ring(i.incident_sla, 'Incident SLA')}</section>`,
    };
  }

  // =====================================================================
  // ACTIVITY TIMELINE (journal + customer updates)
  // =====================================================================
  const check = (text, cls = '') => `<span class="check ${cls}">${cls === 'bad' ? I.warn : I.tick}${esc(text)}</span>`;
  function kciItem(k) {
    const held = k.status === 'Held';
    const writer = k.ai_source === 'openai' ? `Written by KCI Writer (${k.model || 'OpenAI'})` : 'Template (AI demo mode)';
    const verdict = k.review_verdict === 'FIXED' ? check('AI Reviewer: FIXED', 'ai') : k.review_verdict === 'HOLD' ? check('AI Reviewer: HOLD', 'bad') : check(`AI Reviewer: ${k.review_verdict || 'PASS'}`);
    return { ts: k.created_at, k: 'ai', html: `<li data-k="ai"><span class="node" style="background:${held ? 'var(--bad-soft);color:var(--bad)' : 'var(--ai-soft);color:var(--ai)'}">${SPARK}</span>
      <div class="body"><div class="meta"><b>${held ? 'Customer update held for approval' : 'Customer update sent'}</b><span class="pill ai">${esc(eventName(k.event))}</span>
        ${k.trigger === 'proactive' ? '<span class="pill warn">Sent automatically (SLA)</span>' : ''}${k.customer_name && R.type === 'incident' ? `<span>to ${esc(k.customer_name)}</span>` : ''}<span>· ${esc(ago(k.created_at))}</span></div>
        <div class="bubble kci ${held ? 'held' : ''}"><div class="subj">${esc(k.subject)}</div>${esc(k.message)}
          ${k.sms ? `<div class="sms"><b>SMS</b><span>${esc(k.sms)}</span></div>` : ''}
          <div class="checks">${check(writer)}${verdict}${!held ? check('Email + SMS sent') : ''}</div>
          ${k.review_notes ? `<div class="note" style="margin-top:8px">Reviewer: ${esc(k.review_notes)}</div>` : ''}
          ${held ? `<textarea id="held-${k.id}" aria-label="Edit the held update">${esc(k.message)}</textarea>
            <div class="acts" style="margin-top:8px"><button class="btn primary sm" type="button" data-approve="${k.id}">Approve and send</button><span class="note">Edit the text first if needed. It is sent by email and SMS.</span></div>` : ''}
        </div></div></li>` };
  }
  function entryItem(e) {
    const a = e.author_name || e.author_type;
    const aiAuthor = /^AI |Photo AI/.test(a);
    if (e.entry_type === 'event') {
      return { ts: e.created_at, k: aiAuthor ? 'ai' : 'internal', html: `<li class="ev ${aiAuthor ? 'ai' : ''}" data-k="${aiAuthor ? 'ai' : 'internal'}"><span class="node"><i></i></span>
        <div class="body"><div class="event">${aiAuthor ? `<span class="ai-tag">${SPARK}${esc(a)}</span> ` : ''}${esc(e.message)} <span class="muted">· ${esc(ago(e.created_at))}</span></div></div></li>` };
    }
    if (e.entry_type === 'work_note') {
      return { ts: e.created_at, k: 'internal', html: `<li data-k="internal"><span class="node" style="background:#FFF3DE;color:var(--warn)">${I.lock}</span>
        <div class="body"><div class="meta"><b>Internal note</b><span>${aiAuthor ? `<span class="ai-tag">${SPARK}${esc(a)}</span>` : esc(a)} · ${esc(ago(e.created_at))}</span><span class="pill warn">Not visible to customer</span></div>
        <div class="bubble internal">${esc(e.message)}</div></div></li>` };
    }
    if (e.entry_type === 'email') {
      const [, to, subject, body] = e.message.match(/^To: (.*?) \| Subject: (.*?) \| ([\s\S]*)$/) || [null, '', '', e.message];
      return { ts: e.created_at, k: 'customer', html: `<li data-k="customer"><span class="node" style="background:var(--brand-soft);color:var(--brand)">${I.mail}</span>
        <div class="body"><div class="meta"><b>Email sent by ${esc(a)}</b><span>· ${esc(ago(e.created_at))}</span></div>
        <div class="bubble mail"><div class="mh">To ${esc(to)} · <b>${esc(subject)}</b></div>${esc(body)}<div class="checks">${check('Checked by AI Reviewer')}</div></div></div></li>` };
    }
    const customer = e.author_type === 'customer';
    const bot = e.author_type === 'bot';
    const label = customer ? `${esc(a)}${e.entry_type === 'bot_question' ? ' <span class="muted">via Order Assistant</span>' : e.channel === 'Photo' ? ' <span class="muted">sent a photo</span>' : ''}`
      : bot ? `<span class="ai-tag">${SPARK}${esc(a)}</span>` : `Reply to customer <span class="muted">by ${esc(a)}</span>`;
    return { ts: e.created_at, k: 'customer', html: `<li data-k="customer"><span class="node" style="background:${customer ? 'var(--brand-soft);color:var(--brand)' : bot ? 'var(--ai-soft);color:var(--ai)' : 'var(--line-2);color:var(--ink-2)'}">${I.chat}</span>
      <div class="body"><div class="meta"><b>${label}</b><span>· ${esc(ago(e.created_at))}</span>${e.sentiment ? `<span class="pill ${{ angry: 'bad', negative: 'warn', positive: 'ok' }[e.sentiment] || 'grey'}">${esc(e.sentiment)}</span>` : ''}</div>
      <div class="bubble ${customer ? 'customer' : bot ? 'bot' : ''}">${esc(e.message)}</div></div></li>` };
  }
  function timeline(vm) {
    // KCI email/SMS copies are shown inside the update itself
    const items = [
      ...vm.kcis.map(kciItem),
      ...vm.activities.filter((e) => !(['email', 'sms'].includes(e.entry_type) && e.author_type === 'system')).map(entryItem),
    ].sort((a, b) => new Date(b.ts) - new Date(a.ts));
    R.items = items;
    return items.map((x) => x.html).join('') || `<li style="display:block"><div class="empty">${ICON.inbox}<b>No activity yet</b></div></li>`;
  }
  function applyFilter() {
    const m = M(); let shown = 0; let hidden = 0;
    document.querySelectorAll('#timeline > li[data-k]').forEach((li) => {
      const match = m.filter === 'all' || li.dataset.k === m.filter;
      const show = match && shown < m.shown;
      li.style.display = show ? '' : 'none';
      if (show) shown++; else if (match) hidden++;
    });
    $('moreRow').innerHTML = hidden ? `<button class="btn sm" type="button" id="showMore">Show ${hidden} older</button>` : '';
    if (hidden) $('showMore').onclick = () => { m.shown += 50; applyFilter(); };
  }

  // ---------------- composer: reply (Reviewer-checked) or internal note ----------------
  function composer(vm) {
    const m = M();
    if (!vm.customer && m.mode === 'public') m.mode = 'internal';
    return `<div class="composer ${m.mode === 'internal' ? 'internal' : ''}" id="composer">
      <div class="tabs">${vm.customer ? `<button type="button" data-mode="public" class="${m.mode === 'public' ? 'on' : ''}">Reply to customer</button>` : ''}<button type="button" data-mode="internal" class="${m.mode === 'internal' ? 'on' : ''}">Internal note</button></div>
      <textarea id="compText" placeholder="${m.mode === 'internal' ? 'Add an internal note for your team…' : `Write a reply to ${esc(first(vm.customer?.name))}… the AI Reviewer checks it before it is sent.`}"></textarea>
      <div class="err" id="compErr" hidden></div>
      <div class="bar"><small id="compHint">${m.mode === 'internal' ? 'Only your team can see this' : 'Visible to the customer · checked by the AI Reviewer'}</small>
        <div class="acts">${vm.customer ? `<button class="btn ai sm" type="button" id="compSuggest" ${m.mode === 'internal' ? 'hidden' : ''}>${SPARK}Suggest reply</button>` : ''}<button class="btn primary sm" type="button" id="compPost">Post</button></div></div>
    </div>`;
  }
  async function postComposer() {
    const m = M(); const text = $('compText').value.trim(); const err = $('compErr');
    if (!text) return;
    const btn = $('compPost'); btn.disabled = true; err.hidden = true;
    try {
      if (m.mode === 'internal') {
        await api(`/api/ws/records/${R.type}/${R.id}/note`, { message: text, agent_name: W.AGENT.name });
        toast('Internal note added');
      } else {
        const r = await api(`/api/ws/records/${R.type}/${R.id}/reply`, { message: text, agent_name: W.AGENT.name });
        if (r.status === 'Held') { err.innerHTML = `<b>Not sent: the AI Reviewer stopped this reply.</b> ${esc(r.issues.join('; '))}`; err.hidden = false; return; }
        toast(`Reply sent · AI Reviewer: ${r.verdict}`);
      }
      $('compText').value = ''; await reload();
    } catch (e) { err.textContent = e.message; err.hidden = false; }
    finally { if ($('compPost')) $('compPost').disabled = false; }
  }

  // =====================================================================
  // RELATED TABS
  // =====================================================================
  function drawTab(vm) {
    const tab = vm.tabs.find((t) => t.key === M().tab);
    $('tabAct').innerHTML = ''; $('rpane').innerHTML = tab ? tab.draw() : '';
  }
  const table = (head, rows, empty) => rows.length
    ? `<div class="tbl-wrap"><table><thead><tr>${head.map((h) => `<th>${h}</th>`).join('')}</tr></thead><tbody>${rows.join('')}</tbody></table></div>`
    : `<div class="empty">${ICON.inbox}<b>${esc(empty[0])}</b>${esc(empty[1] || '')}</div>`;
  const tabButton = (label, run) => { $('tabAct').innerHTML = `<button class="btn sm" type="button" id="tabBtn">${ICON.plus.replace('<svg', '<svg width="14" height="14"')}${esc(label)}</button>`; setTimeout(() => { if ($('tabBtn')) $('tabBtn').onclick = run; }); };

  function slaTab(slas) {
    return table(['SLA', 'Target', 'Started', 'Progress', 'State'], slas.map((s) => `<tr style="cursor:default"><td>${esc(s.name)}</td><td>${esc(s.target)}</td><td>${esc(when(s.started_at))}</td>
      <td>${slaCell(s)}</td><td>${s.state === 'In Progress' ? (s.breached ? '<span class="pill bad">Breached</span>' : '<span class="pill blue">In progress</span>') : s.state === 'Completed' ? (s.breached ? '<span class="pill bad">Breached</span>' : '<span class="pill ok">Met</span>') : statePill(s.state)}</td></tr>`), ['No SLAs yet']);
  }

  // ----- order tasks -----
  function orderTaskActions(t) {
    const o = R.data;
    if (!['Open', 'In Progress'].includes(t.state) || o.status !== 'In Progress') return [];
    const delay = t.customer_impact ? [{ a: 'delay', label: 'Report delay' }] : [];
    return [...({
      VALIDATION: [{ a: 'approve', label: 'Approve validation' }],
      CPE_DELIVERY: t.milestone === 'Scheduled' ? [{ a: 'deliver', label: 'Mark delivered' }] : [{ a: 'schedule', label: 'Schedule delivery' }],
      ENGINEER_VISIT: t.milestone === 'Scheduled' ? [{ a: 'complete', label: 'Visit completed' }, { a: 'schedule', label: 'Move visit' }] : [{ a: 'schedule', label: 'Book engineer' }],
      ACTIVATION: [{ a: 'activate', label: 'Activate service' }],
    }[t.task_type] || []), ...delay];
  }
  const shortDay = (ts) => new Date(ts).toLocaleDateString('en-GB', { weekday: 'short', day: 'numeric', month: 'short' });
  function planCell(t) {
    if (!t.planned_end) return '<span class="muted">—</span>';
    const same = t.planned_start && shortDay(t.planned_start) === shortDay(t.planned_end);
    const range = t.planned_start && !same ? `${shortDay(t.planned_start)} → ${shortDay(t.planned_end)}` : shortDay(t.planned_end);
    return `${esc(range)}${t.slipped ? `<span class="sub2" style="color:var(--bad)">was ${esc(shortDay(t.original_planned_end))}</span>` : ''}`;
  }
  function orderTasksTab() {
    const o = R.data;
    return table(['Task', 'Name', 'Planned', 'Actual', 'Impact', 'State', 'SLA', 'Next step'], o.tasks.map((t) => {
      const acts = orderTaskActions(t);
      return `<tr data-task="${t.id}"><td class="mono num">${esc(t.task_number)}</td>
        <td>${esc(t.name)}<span class="sub2">${esc(t.assignment_group)}${t.depends_on_number ? ` · after ${esc(t.depends_on_number)}` : ''}${t.milestone ? ` · ${esc(t.milestone)}` : ''}</span></td>
        <td>${planCell(t)}</td>
        <td>${t.closed_at ? esc(when(t.closed_at)) : t.opened_at ? `<span class="muted">started ${esc(shortDay(t.opened_at))}</span>` : '<span class="muted">—</span>'}</td>
        <td>${t.customer_impact ? '<span class="pill blue">Customer</span>' : '<span class="pill grey">Internal</span>'}</td>
        <td>${statePill(t.state)}</td><td>${t.sla ? slaCell(t.sla) : '<span class="muted">Not started</span>'}</td>
        <td>${acts[0] ? `<button class="btn sm ${acts[0].a === 'approve' ? '' : 'primary'}" type="button" data-taskact="${t.id}:${acts[0].a}">${esc(acts[0].label)}</button>` : t.customer_impact && t.state === 'Closed' ? '<span class="ai-tag">AI</span> <span class="muted">customer told</span>' : '<span class="muted">—</span>'}</td></tr>`;
    }), ['No tasks']);
  }
  // Every decision of the AI KCI Decision Agent (at risk / delayed), with its reason
  function decisionsTab() {
    const o = R.data;
    return table(['When', 'Task', 'Situation', 'Decision', 'Why', 'By'], o.decisions.map((d) => {
      const t = o.tasks.find((x) => x.id === d.order_task_id);
      return `<tr style="cursor:default"><td>${esc(when(d.created_at))}</td><td>${esc(t?.name || '—')}<span class="sub2">${esc(d.trigger === 'sla_watcher' ? 'SLA watcher' : 'Agent reported')}</span></td>
        <td>${d.situation === 'DELAYED' ? '<span class="pill bad">Delayed</span>' : '<span class="pill warn">At risk</span>'}</td>
        <td>${d.action === 'SEND_UPDATE' ? `<span class="pill ai">Send ${esc(String(d.update_type || '').toLowerCase())} update</span>` : '<span class="pill grey">No update</span>'}</td>
        <td style="max-width:340px">${esc(d.reason)}${(d.key_points || []).length ? `<span class="sub2">${esc(d.key_points.join(' · '))}</span>` : ''}</td>
        <td>${d.ai_source === 'openai' ? `<span class="ai-tag">${SPARK}${esc(d.model || 'OpenAI')}</span>` : '<span class="muted">Rules (demo)</span>'}</td></tr>`;
    }), ['No decisions yet', 'The AI KCI Decision Agent decides here when a task is at risk of missing its SLA, or is delayed.']);
  }
  async function runOrderTask(taskId, a) {
    const t = R.data.tasks.find((x) => x.id === taskId);
    let extra = {};
    if (t.task_type === 'CPE_DELIVERY' && a === 'schedule') {
      extra = await formDialog({ title: `Schedule ${t.name.toLowerCase()}`, text: (t.details.items || []).join(', '), submit: 'Schedule and tell the customer',
        note: 'The AI writes the customer an update with the date, courier and tracking reference.',
        fields: [{ name: 'date', label: 'Delivery date', type: 'date', value: isoDate(nextWorkingDay()), required: true }] });
      if (!extra) return;
    }
    if (a === 'delay') {
      extra = await formDialog({ title: `Report a delay: ${t.name}`, submit: 'Report delay and tell the customer',
        text: `Planned: ${t.planned_end ? shortDay(t.planned_end) : 'not planned yet'}${t.depends_on_name ? ` · depends on ${t.depends_on_name}` : ''}`,
        note: 'Code calculates the new dates and moves the tasks that depend on this one (e.g. the engineer visit). The AI KCI Decision Agent decides what the customer must hear, and ONE update is sent.',
        fields: [{ name: 'days', label: 'Delayed by (working days)', type: 'number', value: '2', required: true },
          { name: 'reason', label: 'Reason (internal: never shown to the customer)', type: 'textarea', required: true, placeholder: 'e.g. Courier depot backlog after the storm' }] });
      if (!extra) return;
    }
    if (t.task_type === 'ENGINEER_VISIT' && a === 'schedule') {
      const slots = await api(`/api/ws/slots?order_id=${R.id}`);
      extra = await formDialog({ title: t.milestone ? 'Move the engineer visit' : 'Book the engineer visit', submit: 'Book and tell the customer',
        text: t.work_notes ? `Engineer note: ${t.work_notes}` : '', note: 'The AI writes the customer an update with the slot and how to prepare.',
        fields: [{ name: 'slot', label: 'Visit slot', type: 'select', options: slots, required: true }] });
      if (!extra) return;
    }
    if (a === 'complete') {
      extra = await formDialog({ title: 'Engineer visit completed', submit: 'Close task and tell the customer', note: 'Leave empty to use the expected line speed.',
        fields: [{ name: 'speed_mbps', label: 'Speed measured on the day (Mbps)', type: 'number', placeholder: 'e.g. 285' }] });
      if (!extra) return;
      if (!extra.speed_mbps) delete extra.speed_mbps;
    }
    try {
      const r = await api(`/api/ws/order-tasks/${taskId}/action`, { action: a, ...extra });
      if (a === 'delay') {
        const moved = (r.facts.moved || []).map((m) => `${m.task} → ${m.to}`).join(', ');
        toast(`New date ${r.facts.new_date}${moved ? ` · moved: ${moved}` : ''} · AI decided: ${r.decision.update_type?.toLowerCase()} update · customer ${r.kci.status === 'Held' ? 'update HELD for review' : 'told'}`, r.kci.status === 'Held');
      } else toastKci(r.kci, 'Task updated', stageNote(r.stage_decision));
      $('formDlg').open && $('formDlg').close();
      await reload();
    } catch (e) { toast(e.message, true); }
  }
  // The Stage Agent's hold note, while the order is still waiting in that stage for a person
  function stageHold(o) {
    if (!o.can_advance || !o.needs_attention) return null;
    const n = (o.activities || []).find((a) => a.author_name === 'AI Stage Agent' && a.entry_type === 'work_note');
    return n && n.message.startsWith(`Held in ${o.current_stage}:`) ? n.message.replace(/^Held in [^:]+: /, '') : null;
  }
  function toastKci(k, fallback, extra = '') {
    if (!k) { toast(fallback + extra); return; }
    toast((k.status === 'Held' ? `${eventName(k.event)}: update HELD by the AI Reviewer, approve it in the activity` : `Customer told: ${eventName(k.event)} · AI Reviewer ${k.review_verdict}`) + extra, k.status === 'Held');
  }
  // What the AI Stage Agent did after the task closed (null = the stage still has open work)
  const stageNote = (d) => !d ? '' : d.moved
    ? ` · AI Stage Agent moved the order to ${d.to}${d.kci ? ' (final update sent)' : ''}`
    : ` · AI Stage Agent HELD the order in ${d.from}: see the work note`;
  function orderTaskDialog(taskId) {
    const t = R.data.tasks.find((x) => x.id === taskId);
    const d = t.details || {};
    const rows = [['Team', t.assignment_group + (t.assigned_to ? ` · ${t.assigned_to}` : '')], ['State', t.state + (t.milestone ? ` · ${t.milestone}` : '')], ['Stage', t.stage],
      ['Items', (d.items || []).join(', ')], ['Delivery', d.delivery_date ? `${d.delivery_date}, ${d.delivery_window}` : ''], ['Courier', d.courier ? `${d.courier} · ${d.tracking_ref}` : ''],
      ['Delivered', d.delivered_at], ['Visit slot', d.slot], ['Work', d.work], ['Speed measured', d.speed_mbps ? `${d.speed_mbps} Mbps` : ''], ['Activated', d.activated_at],
      ['Planned start', t.planned_start ? when(t.planned_start) : ''], ['Planned end', t.planned_end ? when(t.planned_end) : ''],
      ['Original plan', t.slipped ? when(t.original_planned_end) : ''], ['Actual start', when(t.opened_at)], ['Actual completion', when(t.closed_at)],
      ['Depends on', t.depends_on_number ? `${t.depends_on_number} · ${t.depends_on_name}` : ''], ['Customer impact', t.customer_impact ? 'Yes: the customer is updated' : 'No: internal task'],
      ['Delay reason', t.delay_reason ? `${t.delay_reason} (internal)` : ''],
      ['SLA', t.sla ? `${t.sla.name} · ${t.sla.time_left || (t.sla.breached ? 'breached' : t.sla.state === 'Completed' ? 'met' : t.sla.state)}` : 'Not started']].filter(([, v]) => v);
    const acts = orderTaskActions(t);
    infoDialog({ title: `${t.task_number} · ${t.name}`, html: `<dl class="task">${rows.map(([k, v]) => `<dt>${esc(k)}</dt><dd>${esc(v)}</dd>`).join('')}</dl>
      ${t.work_notes ? `<div class="bubble internal" style="margin-top:12px"><b>Internal notes:</b> ${esc(t.work_notes)}</div>` : ''}`,
      actions: acts.map((x) => ({ label: x.label, primary: x.a !== 'approve', run: () => { $('formDlg').close(); runOrderTask(t.id, x.a); } })) });
  }

  function orderIncidentsTab() {
    const o = R.data;
    if (['In Progress', 'On Hold'].includes(o.status)) tabButton('Raise incident', raiseIncident);
    return table(['Incident', 'Description', 'Priority', 'State', 'Raised'], o.incidents.map((i) => `<tr data-href="${W.recordHash('incident', i.id)}"><td class="mono num">${esc(i.incident_number)}</td>
      <td>${esc(i.description)}${i.parent_id ? '<span class="sub2">Part of a major incident</span>' : ''}</td><td>${priPill(i.priority)}</td><td>${statePill(i.state)}</td><td>${esc(when(i.created_at))}</td></tr>`),
    ['No incidents on this order', 'Raise one if fulfilment hits a problem. The customer gets an AI-written delay update.']);
  }
  function orderCasesTab() {
    const o = R.data;
    tabButton('New case', () => W.openNewCase({ order_number: o.order_number }));
    return table(['Case', 'Type', 'Title', 'State', 'Opened'], o.cases.map((c) => `<tr data-href="${W.recordHash('case', c.id)}"><td class="mono num">${esc(c.case_number)}</td><td>${esc(c.type)}</td>
      <td>${esc(c.short_description)}${c.origin === 'Order Assistant' ? '<span class="sub2"><span class="ai-tag">AI</span> from Order Assistant</span>' : ''}</td><td>${statePill(c.state)}</td><td>${esc(when(c.created_at))}</td></tr>`),
    ['No cases for this order', 'The Order Assistant opens one automatically if the customer needs a human.']);
  }

  // ----- case tasks -----
  function caseTasksTab() {
    const c = R.data;
    if (['New', 'Open'].includes(c.state)) tabButton('Add task', addCaseTask);
    return table(['Task', 'What', 'Team', 'State', 'SLA', ''], c.tasks.map((t) => {
      const open = ['Open', 'In Progress'].includes(t.state);
      return `<tr data-ctask="${t.id}"><td class="mono num">${esc(t.task_number)}</td><td>${esc(t.short_description)}${t.customer_note ? '<span class="sub2"><span class="ai-tag">AI</span> customer told when closed</span>' : ''}</td>
        <td>${esc(t.assignment_group)}${t.assigned_to ? `<span class="sub2">${esc(t.assigned_to)}</span>` : ''}</td><td>${statePill(t.state)}</td><td>${slaCell(t.sla)}</td>
        <td>${open ? `<div class="acts">${t.state === 'Open' ? `<button class="btn sm" type="button" data-cstate="${t.id}:In Progress">Start</button>` : ''}<button class="btn sm primary" type="button" data-cstate="${t.id}:Closed">Close</button></div>` : ''}</td></tr>`;
    }), ['No tasks', 'Add a task to hand work to another team. A case cannot be resolved while a task is open.']);
  }
  async function setCaseTask(id, state) {
    try {
      const r = await api(`/api/ws/case-tasks/${id}/state`, { state });
      toastKci(r.kci, `Task ${state.toLowerCase()}`); await reload();
    } catch (e) { toast(e.message, true); }
  }
  async function addCaseTask() {
    const opts = await W.caseOptions();
    await formDialog({ title: 'Add a task', submit: 'Add task', note: 'If you fill in the customer note, the customer gets an AI-written progress update when the task is closed.',
      fields: [{ name: 'short_description', label: 'What must the team do?', type: 'textarea', required: true },
        { name: 'assignment_group', label: 'Team', type: 'select', options: opts.teams, required: true },
        { name: 'assigned_to', label: 'Assigned to (optional)' },
        { name: 'customer_note', label: 'Customer note (optional)', placeholder: 'e.g. the duplicate payment has been refunded to your card' }],
      onSubmit: async (v) => { await api(`/api/ws/cases/${R.id}/tasks`, v); toast('Task added'); await reload(); } });
  }

  // ----- incident tasks + affected customers -----
  function incidentTasksTab() {
    const i = R.data;
    if (['New', 'Open'].includes(i.state)) tabButton('Add task', addIncidentTask);
    return table(['Task', 'What', 'Team', 'State', ''], i.tasks.map((t) => `<tr style="cursor:default"><td class="mono num">${esc(t.task_number)}</td>
      <td>${esc(t.short_description)}${t.customer_note ? `<span class="sub2">Customer note: ${esc(t.customer_note)}</span>` : ''}</td>
      <td>${esc(t.assignment_group)}${t.assigned_to ? `<span class="sub2">${esc(t.assigned_to)}</span>` : ''}</td><td>${statePill(t.state)}</td>
      <td>${t.state !== 'Done' ? `<div class="acts">${t.state === 'Open' ? `<button class="btn sm" type="button" data-istate="${t.id}:In Progress">Start</button>` : ''}<button class="btn sm primary" type="button" data-istate="${t.id}:Done">Done</button></div>` : ''}</td></tr>`),
    ['No tasks', i.parent_id ? 'Tasks are managed on the major incident.' : 'Split the fix into tasks, one team each. When a task is done, affected customers are told.']);
  }
  async function setIncidentTask(id, state) {
    try {
      const r = await api(`/api/tasks/${id}/state`, { state });
      toast(state === 'Done' ? `Task done · ${r.notified} customer${r.notified === 1 ? '' : 's'} told${r.held ? ` · ${r.held} held for review` : ''}` : 'Task started');
      await reload();
    } catch (e) { toast(e.message, true); }
  }
  async function addIncidentTask() {
    const opts = await W.caseOptions();
    await formDialog({ title: 'Add an incident task', submit: 'Add task', note: 'When the task is done, every affected customer gets an AI-written progress update.',
      fields: [{ name: 'short_description', label: 'What must the team do? (internal)', type: 'textarea', required: true },
        { name: 'customer_note', label: 'What may the customer be told when it is done?', placeholder: 'e.g. the damaged cable has been replaced' },
        { name: 'assignment_group', label: 'Team', type: 'select', options: opts.teams, value: R.data.assignment_group, required: true },
        { name: 'assigned_to', label: 'Assigned to (optional)' }],
      onSubmit: async (v) => { await api(`/api/incidents/${R.id}/tasks`, v); toast('Task added'); await reload(); } });
  }
  function affectedTab() {
    const i = R.data;
    return table(['Incident', 'Customer', 'Order', 'Plan', 'State', 'Last update'], i.children.map((c) => `<tr data-href="${W.recordHash('order', c.order_id)}"><td class="mono num">${esc(c.incident_number)}</td>
      <td><div class="who">${avatar(c.customer_name)}<div><b>${esc(c.customer_name)}</b></div></div></td><td class="mono">${esc(c.order_number)}</td><td>${esc(c.product_name)}</td><td>${statePill(c.state)}</td>
      <td>${c.last_kci_status === 'Held' ? '<span class="pill bad">Held</span>' : c.last_kci_status ? `<span class="pill ok">Sent</span> <span class="muted">${esc(c.last_verdict || '')}</span>` : '<span class="muted">—</span>'}</td></tr>`), ['No linked orders']);
  }

  // =====================================================================
  // HEADER ACTIONS
  // =====================================================================
  async function act(url, body, done) {
    try { const r = await api(url, body); toast(typeof done === 'function' ? done(r) : done); await reload(); return r; }
    catch (e) { toast(e.message, true); return null; }
  }
  const advanceOrder = () => act(`/api/ws/orders/${R.id}/advance`, {}, (r) => r.kci ? `Order completed · final update ${r.kci.status === 'Held' ? 'held for review' : 'sent to the customer'}` : `Moved to ${r.to}${r.to === 'Fulfilment' ? ' · first task opened' : ''}`);
  async function advanceCase() {
    const c = R.data;
    if (c.next_stage === 'Resolved') {
      await formDialog({ title: `Resolve ${c.case_number}`, submit: 'Resolve and tell the customer', note: 'The AI writes the customer an update with this resolution.',
        fields: [{ name: 'resolution', label: 'What was done for the customer?', type: 'textarea', required: true }],
        onSubmit: async (v) => { const r = await api(`/api/ws/cases/${c.id}/advance`, { ...v, agent_name: W.AGENT.name }); toastKci(r.kci, 'Resolved'); await reload(); } });
      return;
    }
    act(`/api/ws/cases/${c.id}/advance`, { agent_name: W.AGENT.name }, (r) => `Moved to ${r.to}`);
  }
  async function advanceIncident() {
    const i = R.data;
    if (i.state === 'New') return act(`/api/ws/incidents/${i.id}/start`, { agent_name: W.AGENT.name }, 'Work started');
    if (i.state === 'Resolved') return act(`/api/ws/incidents/${i.id}/close`, {}, 'Incident closed');
    await formDialog({ title: `Resolve ${i.incident_number}`, submit: i.is_major ? 'Resolve and tell every customer' : 'Resolve and tell the customer',
      note: i.is_major ? `All ${i.children.length} linked incidents are resolved too; each customer gets a personal update.` : 'The order goes back in progress and the customer gets an AI-written update.',
      fields: [{ name: 'resolution', label: 'Resolution', type: 'textarea', required: true }],
      onSubmit: async (v) => {
        const r = await api(`/api/incidents/${i.id}/resolve`, v);
        toast(i.is_major ? `Resolved · ${r.resolved} incidents · ${r.notified} customers told` : `Resolved · customer ${r.kci?.status === 'Held' ? 'update held for review' : 'told'}`);
        await reload();
      } });
  }
  async function raiseIncident() {
    const opts = await W.caseOptions();
    await formDialog({ title: `Raise an incident on ${R.data.order_number}`, submit: 'Raise and tell the customer',
      note: 'The order goes on hold; the customer gets an AI-written delay update, checked by the AI Reviewer.',
      fields: [{ name: 'description', label: 'What went wrong? (internal)', type: 'textarea', required: true, placeholder: 'e.g. Engineer could not access the building' },
        { name: 'priority', label: 'Priority', type: 'select', options: ['P3', 'P2', 'P1', 'P4'] },
        { name: 'assignment_group', label: 'Team', type: 'select', options: opts.teams, value: 'Field Engineering' },
        { name: 'testLeak', label: 'Reviewer test: make the draft leak an internal note (to show the AI Reviewer holding it)', type: 'checkbox' }],
      onSubmit: async (v) => {
        const r = await api(`/api/ws/orders/${R.id}/incidents`, v);
        toast(`${r.incident.incident_number} raised · ${r.kci?.status === 'Held' ? 'delay update HELD by the AI Reviewer' : 'customer told about the delay'}`, r.kci?.status === 'Held');
        await reload();
      } });
  }
  async function cancelOrder() {
    await formDialog({ title: `Cancel ${R.data.order_number}`, submit: 'Cancel order', danger: true, note: 'Open tasks and SLAs are cancelled.',
      fields: [{ name: 'reason', label: 'Reason', type: 'textarea', required: true }],
      onSubmit: async (v) => { await api(`/api/ws/orders/${R.id}/cancel`, v); toast('Order cancelled'); await reload(); } });
  }
  async function cancelCase() {
    await formDialog({ title: `Cancel ${R.data.case_number}`, submit: 'Cancel', danger: true, note: 'Open tasks and SLAs are cancelled. No update is sent to the customer.',
      fields: [{ name: 'reason', label: 'Reason', type: 'textarea', required: true }],
      onSubmit: async (v) => { await api(`/api/ws/cases/${R.id}/cancel`, v); toast('Cancelled'); await reload(); } });
  }
  async function save() {
    const fields = {};
    document.querySelectorAll('.rail .inl[data-f]').forEach((el) => { fields[el.dataset.f] = el.value; });
    const url = R.type === 'order' ? `/api/ws/orders/${R.id}` : `/api/ws/cases/${R.id}`;
    try { await api(url, fields, 'PATCH'); toast('Saved'); await reload(); } catch (e) { toast(e.message, true); }
  }

  // =====================================================================
  // AI SUMMARY
  // =====================================================================
  function drawSummary() {
    const m = M(); const box = $('sumPanel');
    if (!m.showSummary) { box.innerHTML = ''; return; }
    if (!m.summary) { box.innerHTML = `<div class="summary"><span class="ai-tag">${SPARK}AI summary</span><div class="grid"><div class="skel" style="height:60px"></div><div class="skel" style="height:60px"></div><div class="skel" style="height:60px"></div></div></div>`; return; }
    const s = m.summary;
    box.innerHTML = `<div class="summary"><span class="ai-tag">${SPARK}AI summary</span>
      <div class="grid"><div><h4>Where things stand</h4><p>${esc(s.summary)}</p><p class="note" style="margin-top:6px">${esc(s.customer_wants)}</p></div>
        <div><h4>Next best action</h4><p>${esc(s.next_best_action)}</p></div>
        <div><h4>Risk</h4><p><span class="pill ${{ low: 'ok', medium: 'warn', high: 'bad' }[s.risk_level]}">${esc(s.risk_level)}</span> ${esc(s.risk_reason)}</p></div></div>
      <div class="foot"><span>Based on ${s.based_on?.activities ?? 0} activities, ${s.based_on?.tasks ?? 0} tasks and ${s.based_on?.updates ?? 0} customer updates. Internal notes are used for understanding only.</span>
        <span class="acts">${s.suggested_reply ? '<button class="btn sm" type="button" id="useReply">Use suggested reply</button>' : ''}<button class="btn sm" type="button" id="sumRefresh">Refresh</button>
        <span>${s.ai_source === 'openai' ? `OpenAI · ${esc(s.model)}${s.latency_ms ? ` · ${(s.latency_ms / 1000).toFixed(1)} s` : ''}` : 'Rules (AI demo mode)'}</span></span></div></div>`;
    if ($('useReply')) $('useReply').onclick = () => { setMode('public'); $('compText').value = s.suggested_reply; $('compText').focus(); $('compText').scrollIntoView({ block: 'center' }); };
    $('sumRefresh').onclick = () => loadSummary(true);
  }
  async function loadSummary(force) {
    const m = M();
    if (m.summary && !force) { drawSummary(); return; }
    m.summary = null; m.showSummary = true; drawSummary();
    try { m.summary = await api(`/api/ws/records/${R.type}/${R.id}/summary`, {}); }
    catch (e) { m.showSummary = false; toast(e.message, true); }
    drawSummary();
  }

  // =====================================================================
  // COPILOT
  // =====================================================================
  function drawChat() {
    const m = M(); const body = $('cpBody');
    body.innerHTML = m.chat.length ? m.chat.map((c) => c.role === 'agent' ? `<div class="cp-q">${esc(c.text)}</div>`
      : c.typing ? '<div class="cp-a typing"><i></i><i></i><i></i></div>' : `<div class="cp-a ${c.rules ? 'rules' : ''}">${esc(c.text)}</div>`).join('')
      : `<p class="cp-hint">Ask anything about this record: what is blocking it, the SLA risk, or a reply for the customer. Answers come only from this record's data.</p>`;
    body.scrollTop = body.scrollHeight;
  }
  async function ask(q) {
    q = String(q || '').trim(); if (!q) return;
    const m = M(); const history = m.chat.filter((c) => !c.typing).map((c) => ({ role: c.role, text: c.text }));
    m.chat.push({ role: 'agent', text: q }, { role: 'copilot', typing: true }); $('cpIn').value = ''; drawChat();
    try {
      const r = await api(`/api/ws/records/${R.type}/${R.id}/copilot`, { question: q, history });
      m.chat.pop(); m.chat.push({ role: 'copilot', text: r.answer, rules: r.ai_source !== 'openai' });
    } catch (e) { m.chat.pop(); m.chat.push({ role: 'copilot', text: e.message, rules: true }); }
    if ($('cpBody')) drawChat();
  }

  // =====================================================================
  // DRAFT EMAIL (Write with AI -> Check and send through the AI Reviewer)
  // =====================================================================
  function openMail() {
    const vm = model();
    $('mailRef').textContent = `${vm.ref} · ${vm.title}`;
    $('mTo').value = vm.customer?.email || ''; $('mSubj').value = ''; $('mBody').value = ''; $('mIntent').value = '';
    $('mReview').hidden = true; $('mailDlg').showModal();
  }
  $('mAi').onclick = async () => {
    const btn = $('mAi'); btn.disabled = true; btn.lastChild.textContent = 'Writing…'; $('mReview').hidden = true;
    try {
      const d = await api(`/api/ws/records/${R.type}/${R.id}/email/draft`, { intent: $('mIntent').value.trim() || undefined });
      if (!$('mTo').value.trim()) $('mTo').value = d.to.join(', ');
      $('mSubj').value = d.subject;
      const ta = $('mBody'); ta.value = ''; let i = 0;
      await new Promise((done) => { const t = setInterval(() => { ta.value = d.body.slice(0, (i += 8)); if (i >= d.body.length) { clearInterval(t); done(); } }, 12); });
      if (d.ai_source !== 'openai') { $('mReview').className = 'review bad'; $('mReview').style.background = 'var(--warn-soft)'; $('mReview').style.color = 'var(--warn)'; $('mReview').innerHTML = `${I.warn}<span>AI demo mode: this draft is a template. Add the OpenAI key for AI-written emails.</span>`; $('mReview').hidden = false; }
    } catch (e) { toast(e.message, true); }
    finally { btn.disabled = false; btn.lastChild.textContent = 'Write with AI'; }
  };
  $('mailForm').addEventListener('submit', async (e) => {
    e.preventDefault();
    const btn = $('mSend'); btn.disabled = true; btn.textContent = 'Checking…';
    const rv = $('mReview'); rv.style.background = ''; rv.style.color = '';
    try {
      const r = await api(`/api/ws/records/${R.type}/${R.id}/email/send`, { to: $('mTo').value, subject: $('mSubj').value, body: $('mBody').value, agent_name: W.AGENT.name });
      if (r.status === 'Held') {
        rv.className = 'review bad'; rv.innerHTML = `${I.warn}<span><b>Not sent: the AI Reviewer stopped this email.</b> ${esc(r.issues.join('; '))}</span>`; rv.hidden = false; return;
      }
      $('mailDlg').close(); toast(`Email sent · AI Reviewer: ${r.verdict}${r.verdict === 'FIXED' ? ' (corrected before sending)' : ''}`); await reload();
    } catch (ex) { rv.className = 'review bad'; rv.innerHTML = `${I.warn}<span>${esc(ex.message)}</span>`; rv.hidden = false; }
    finally { btn.disabled = false; btn.textContent = 'Check and send'; }
  });

  // =====================================================================
  // SHARED DIALOGS
  // =====================================================================
  // formDialog: resolves with the values (or null on cancel). With onSubmit, errors stay inside the dialog.
  function formDialog({ title, text = '', note = '', submit = 'Save', danger = false, fields = [], onSubmit }) {
    return new Promise((resolve) => {
      const dlg = $('formDlg'); const form = $('formDlgForm');
      $('fdTitle').textContent = title; $('fdText').textContent = text; $('fdNote').textContent = note;
      $('fdSubmit').textContent = submit; $('fdSubmit').className = `btn ${danger ? 'danger' : 'primary'}`; $('fdSubmit').hidden = false;
      $('fdBody').innerHTML = fields.map((f) => f.type === 'checkbox'
        ? `<label class="f" style="display:flex;gap:8px;align-items:flex-start;font-weight:500"><input type="checkbox" name="${f.name}" style="width:auto;margin-top:3px"> ${esc(f.label)}</label>`
        : `<label class="f">${esc(f.label)}${f.type === 'textarea' ? `<textarea name="${f.name}" ${f.required ? 'required' : ''} placeholder="${esc(f.placeholder || '')}">${esc(f.value || '')}</textarea>`
          : f.type === 'select' ? `<select name="${f.name}" ${f.required ? 'required' : ''}>${f.options.map((o) => `<option ${o === f.value ? 'selected' : ''}>${esc(o)}</option>`).join('')}</select>`
          : `<input name="${f.name}" type="${f.type || 'text'}" value="${esc(f.value || '')}" placeholder="${esc(f.placeholder || '')}" ${f.required ? 'required' : ''} autocomplete="off">`}</label>`).join('')
        + '<div class="err" id="fdErr" hidden></div>';
      let settled = false;
      const finish = (v) => { if (!settled) { settled = true; resolve(v); } };
      form.onsubmit = async (e) => {
        e.preventDefault();
        const v = {};
        fields.forEach((f) => { const el = form.elements[f.name]; v[f.name] = f.type === 'checkbox' ? el.checked : el.value.trim(); });
        if (!onSubmit) { dlg.close(); finish(v); return; }
        $('fdSubmit').disabled = true;
        try { await onSubmit(v); dlg.close(); finish(v); }
        catch (ex) { $('fdErr').textContent = ex.message; $('fdErr').hidden = false; }
        finally { $('fdSubmit').disabled = false; }
      };
      // A late "close" event from the dialog shown just before (e.g. task details -> Report delay) must not cancel
      // this one: the event arrives after this dialog is already open again, so only a real close counts.
      dlg.onclose = () => { if (!dlg.open) finish(null); };
      dlg.showModal();
      form.querySelector('input:not([type=checkbox]), textarea, select')?.focus();
    });
  }
  // infoDialog: read-only details with optional action buttons
  function infoDialog({ title, html, actions = [] }) {
    $('fdTitle').textContent = title; $('fdText').textContent = ''; $('fdNote').innerHTML = '';
    $('fdBody').innerHTML = html;
    $('fdSubmit').hidden = true;
    $('fdNote').innerHTML = actions.map((a, i) => `<button class="btn ${a.primary ? 'primary' : ''} sm" type="button" data-info="${i}">${esc(a.label)}</button>`).join(' ');
    $('fdNote').querySelectorAll('[data-info]').forEach((b) => (b.onclick = actions[Number(b.dataset.info)].run));
    $('formDlgForm').onsubmit = (e) => e.preventDefault();
    $('formDlg').onclose = null;
    $('formDlg').showModal();
  }

  // =====================================================================
  // WIRING
  // =====================================================================
  function setMode(mode) {
    M().mode = mode;
    const c = $('composer'); if (!c) return;
    c.classList.toggle('internal', mode === 'internal');
    c.querySelectorAll('.tabs button').forEach((b) => b.classList.toggle('on', b.dataset.mode === mode));
    $('compHint').textContent = mode === 'internal' ? 'Only your team can see this' : 'Visible to the customer · checked by the AI Reviewer';
    $('compText').placeholder = mode === 'internal' ? 'Add an internal note for your team…' : 'Write a reply… the AI Reviewer checks it before it is sent.';
    if ($('compSuggest')) $('compSuggest').hidden = mode === 'internal';
    $('compErr').hidden = true;
  }
  function wire(vm) {
    $('sumBtn').onclick = () => { const m = M(); if (m.showSummary && m.summary) { m.showSummary = false; drawSummary(); } else loadSummary(false); };
    if ($('mailBtn')) $('mailBtn').onclick = openMail;
    if ($('saveBtn')) $('saveBtn').onclick = save;
    if ($('advBtn')) $('advBtn').onclick = () => vm.advance.run();
    const menu = $('moreMenu');
    if (menu) {
      menu.querySelector('button').onclick = (e) => { e.stopPropagation(); menu.classList.toggle('open'); };
      menu.querySelectorAll('[data-more]').forEach((b) => (b.onclick = () => { menu.classList.remove('open'); vm.more[Number(b.dataset.more)].run(); }));
    }
    $('actFilter').onclick = (e) => { const b = e.target.closest('button'); if (!b) return; M().filter = b.dataset.f; M().shown = 25;
      $('actFilter').querySelectorAll('button').forEach((x) => x.classList.toggle('on', x === b)); applyFilter(); };
    $('composer').querySelectorAll('.tabs button').forEach((b) => (b.onclick = () => setMode(b.dataset.mode)));
    $('compPost').onclick = postComposer;
    $('compText').addEventListener('keydown', (e) => { if (e.key === 'Enter' && (e.ctrlKey || e.metaKey)) postComposer(); });
    if ($('compSuggest')) $('compSuggest').onclick = async () => {
      const b = $('compSuggest'); b.disabled = true;
      try { const s = M().summary || (M().summary = await api(`/api/ws/records/${R.type}/${R.id}/summary`, {})); $('compText').value = s.suggested_reply || ''; $('compText').focus(); }
      catch (e) { toast(e.message, true); } finally { b.disabled = false; }
    };
    $('rtabs').onclick = (e) => { const b = e.target.closest('button.t'); if (!b) return; M().tab = b.dataset.t;
      $('rtabs').querySelectorAll('button.t').forEach((x) => x.classList.toggle('on', x === b)); drawTab(vm); };
    document.querySelectorAll('.cp-chips button').forEach((b) => (b.onclick = () => ask(b.textContent)));
    $('cpSend').onclick = () => ask($('cpIn').value);
    $('cpIn').addEventListener('keydown', (e) => { if (e.key === 'Enter') ask($('cpIn').value); });
    document.querySelectorAll('.rail .inl').forEach((el) => el.addEventListener('input', () => el.classList.add('dirty')));
  }
  // clicks inside the record page (rows, task buttons, held updates)
  document.addEventListener('click', async (e) => {
    if (!e.target.closest('#page') || !R.data) return;
    const ta = e.target.closest('[data-taskact]'); if (ta) { e.stopPropagation(); const [id, a] = ta.dataset.taskact.split(':'); runOrderTask(Number(id), a); return; }
    const cs = e.target.closest('[data-cstate]'); if (cs) { const [id, s] = cs.dataset.cstate.split(':'); setCaseTask(Number(id), s); return; }
    const is = e.target.closest('[data-istate]'); if (is) { const [id, s] = is.dataset.istate.split(':'); setIncidentTask(Number(id), s); return; }
    const ap = e.target.closest('[data-approve]');
    if (ap) {
      ap.disabled = true;
      try { await api(`/api/kcis/${ap.dataset.approve}/approve`, { message: $(`held-${ap.dataset.approve}`).value }); toast('Approved and sent to the customer'); await reload(); }
      catch (ex) { toast(ex.message, true); ap.disabled = false; }
      return;
    }
    const tr = e.target.closest('tr[data-task]'); if (tr && !e.target.closest('button, a')) { orderTaskDialog(Number(tr.dataset.task)); return; }
    if (!e.target.closest('.menu')) document.querySelectorAll('.menu.open').forEach((m) => m.classList.remove('open'));
  }, true);

  W.formDialog = formDialog;   // shared with workspace.js (e.g. Reset demo data)

  // case options (categories, teams) for the editable details
  W.caseOptions().then((o) => { CASE_OPTS = o; }).catch(() => {});
})();
