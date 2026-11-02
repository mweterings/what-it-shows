<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem merged into a ticket. causes.ts places a report by tenant and form;
  // the desk's one-day list (desk-notes.json) is a third problem, not a footnote, so it is
  // taken out of whichever cause it was filed under and counted on its own.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  const LABEL: Record<string, string> = {
    parser: 'Shared parser reads day-first dates month first',
    hint: 'booking-v3 form hints MM/DD/YYYY, reads DD/MM',
    'one-day': 'Date moves by one day (time zone)',
    unplaced: 'No tenant on file',
  };

  const rows = $derived.by(() => {
    const all = uniqueById(reports as Report[]);
    const merged = all.filter((r) => r.mergedInto === ticket);
    const oneDay = new Set<string>(notes.oneDayMove);
    const groups = groupByCause(merged.filter((r) => !oneDay.has(r.id)), tenantOf).map((g) => ({
      key: g.cause as string,
      n: g.reports.length,
      tenants: g.tenants,
      seen: `${g.firstSeen} → ${g.lastSeen}`,
      mistagged: g.reports.filter((r) => r.meta.intakeTag !== g.cause).length,
    }));
    const od = merged.filter((r) => oneDay.has(r.id));
    const days = od.map((r) => r.receivedAt.slice(0, 10)).sort();
    if (od.length)
      groups.push({ key: 'one-day', n: od.length, tenants: new Set(od.map((r) => r.tenant)).size, seen: `${days[0]} → ${days[days.length - 1]}`, mistagged: od.length });
    return { rows: groups, total: merged.length, raw: (reports as Report[]).filter((r) => r.mergedInto === ticket).length };
  });
</script>

<p class="sum">
  <strong>{rows.total}</strong> reports in #{ticket} ({rows.raw - rows.total} export rows were repeats) ·
  <strong>{rows.rows.length}</strong> problems
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>intake tag wrong</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows.rows as r (r.key)}
      <tr>
        <td>{LABEL[r.key] ?? r.key} <code>{r.key}</code></td>
        <td>{r.n}</td>
        <td>{r.tenants}</td>
        <td>{r.seen}</td>
        <td>{r.mistagged}</td>
        <td>{#if issues[r.key]}<a href={issues[r.key]}>{issues[r.key].split('/').slice(-3).join(' ').replace(' issues ', '#')}</a>{:else}—{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
