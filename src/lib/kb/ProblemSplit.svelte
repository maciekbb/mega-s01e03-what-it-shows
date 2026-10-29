<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { PROBLEMS, causeOf, uniqueById, type Problem, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `issues` maps each problem to the ticket it was filed as.
  let { ticket, issues }: { ticket: number; issues: Partial<Record<Problem, string>> } = $props();

  const rows = $derived(reports as Report[]);
  const merged = $derived(uniqueById(rows).filter((r) => r.mergedInto === ticket));
  const repeats = $derived(rows.filter((r) => r.mergedInto === ticket).length - merged.length);

  const groups = $derived.by(() => {
    const keys = [...PROBLEMS, 'unplaced'] as const;
    return keys
      .map((problem) => {
        // Counted the desk's way: by tenant, as CauseTable does. The one-day moves stay counted where the
        // desk files them, and also make up their own problem.
        const rs =
          problem === 'one-day-move'
            ? merged.filter((r) => notes.oneDayMove.includes(r.id))
            : merged.filter((r) => (causeOf(r, tenantOf(r)) ?? 'unplaced') === problem);
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        const count = (f: (r: Report) => string) =>
          Object.entries(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [f(r)]: (n[f(r)] ?? 0) + 1 }), {}))
            .sort((a, b) => b[1] - a[1])
            .map(([k, n]) => `${k} ×${n}`)
            .join(' · ');
        return {
          problem,
          reports: rs.length,
          taggedAs: count((r) => r.meta.intakeTag),
          channels: count((r) => r.channel),
          tenants: new Set(rs.map((r) => r.tenant)).size,
          seen: rs.length ? `${days[0]} → ${days[days.length - 1]}` : '',
        };
      })
      .filter((g) => g.reports > 0);
  });
  const placed = $derived(merged.filter((r) => causeOf(r, tenantOf(r)) !== null).length);
  const moves = $derived(groups.find((g) => g.problem === 'one-day-move'));
  const filedUnder = $derived(
    [...new Set(merged.filter((r) => notes.oneDayMove.includes(r.id)).map((r) => causeOf(r, tenantOf(r)) ?? 'unplaced'))].join(', '),
  );
  const issueOf = (p: string) => issues[p as Problem];
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{groups.length}</strong> problems · {placed === merged.length ? 'every report placed by tenant' : `${merged.length - placed} reports not placed`}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>ticket</th><th>tagged by intake as</th><th>channels</th><th>tenants</th><th>seen</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code></td>
        <td><strong>{g.reports}</strong></td>
        <td>{#if issueOf(g.problem)}<a href={issueOf(g.problem)}>{issueOf(g.problem)?.replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
        <td class="small">{g.taggedAs}</td>
        <td class="small">{g.channels}</td>
        <td>{g.tenants}</td>
        <td class="small">{g.seen}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if moves}
  <p class="note">The {moves.reports} one-day moves are also counted under <code>{filedUnder}</code>, where the desk's view files them by tenant.</p>
{/if}

{#if repeats}
  <p class="note">The export lists {repeats} reports twice (overlapping pages). Each is counted once.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .small { color: var(--muted); font-size: 0.85em; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
