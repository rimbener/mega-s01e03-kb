<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem the reports in a ticket show. The cause comes from causes.ts (tenant, form,
  // channel); reports the desk read as a one-day move (desk-notes.json) are a problem of their own,
  // so they are taken out of the cause they were filed under, not footnoted under it.
  let { ticket, filed = {} }: { ticket: number; filed?: Record<string, string> } = $props();

  const LABEL: Record<string, string> = {
    parser: 'shared parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD, form reads DD/MM',
    'one-day': 'date moves by one day, month stays right',
    unplaced: 'no tenant on file: cause unknown',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rest = $derived(merged.filter((r) => !notes.oneDayMove.includes(r.id)));
  const rows = $derived(
    [
      ...groupByCause(rest, tenantOf).map((g) => ({ key: g.cause as string, reports: g.reports })),
      ...(oneDay.length ? [{ key: 'one-day', reports: oneDay }] : []),
    ].map((row) => {
      const days = row.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        ...row,
        tenants: new Set(row.reports.map((r) => r.tenant)).size,
        first: days[0],
        last: days[days.length - 1],
      };
    }),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} · <strong>{rows.length}</strong> problems ·
  <strong>{rows.reduce((n, r) => n + r.reports.length, 0)}</strong> placed
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr>
        <td><code>{r.key}</code> · {LABEL[r.key] ?? r.key}</td>
        <td>{r.reports.length}</td>
        <td>{r.tenants}</td>
        <td>{r.first} → {r.last}</td>
        <td>{#if filed[r.key]}<a href={filed[r.key]}>{filed[r.key].replace(/^.*\/issues\//, '#')}</a>{:else}<span class="none">none yet</span>{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .none { color: var(--warn); font-weight: 600; }
</style>
