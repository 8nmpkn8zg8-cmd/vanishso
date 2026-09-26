<script lang="ts">
  import { onMount } from "svelte";

  interface NoteEntry {
    id: string;
    url: string;
    mode: string;
    exp: string;
    createdAt: number;
  }

  let notes: NoteEntry[] = [];

  function formatDate(ts: number): string {
    return new Date(ts).toLocaleString("en-US", {
      month: "short",
      day: "2-digit",
      year: "numeric",
      hour: "numeric",
      minute: "2-digit",
      hour12: true,
    });
  }

  function formatExp(exp: string): string {
    const labels: Record<string, string> = {
      viewing: "after viewing",
      "1h": "1 hour",
      "24h": "24 hours",
      "7d": "7 days",
      "30d": "30 days",
    };
    return labels[exp] ?? exp;
  }

  function clearHistory() {
    try {
      localStorage.removeItem("vanishso_notes");
    } catch {
      // ignore
    }

    notes = [];
  }

  onMount(() => {
    try {
      notes = JSON.parse(localStorage.getItem("vanishso_notes") ?? "[]");
    } catch {
      notes = [];
    }
  });
</script>

<div class="flex flex-col w-full max-w-2xl px-10 mt-6 font-geist">
  <div class="flex items-center justify-between mb-6">
    <h2 class="text-white font-clash font-semibold text-xl">Notes created</h2>
    {#if notes.length > 0}
      <button
        on:click={clearHistory}
        class="text-primary hover:text-white transition-colors text-sm font-medium"
      >
        Clear history
      </button>
    {/if}
  </div>

  {#if notes.length === 0}
    <p class="text-primary text-sm font-medium">
      You haven't created any notes yet.
      <a href="/" class="text-orchid hover:text-orchid-100 transition-colors">Create one</a>.
    </p>
  {:else}
    <div class="flex flex-col divide-y divide-primary/10">
      {#each notes as note}
        <div class="py-4 flex flex-col gap-3">
          <div>
            <p class="text-primary/60 text-xs font-medium uppercase tracking-wider mb-0.5">Link</p>
            <a
              href={note.url}
              class="text-sm font-mono text-primary hover:text-white transition-colors break-all"
            >
              {note.url}
            </a>
          </div>
          <div class="flex gap-8">
            <div>
              <p class="text-primary/60 text-xs font-medium uppercase tracking-wider mb-0.5">Mode</p>
              <p class="text-sm text-primary">{note.mode}</p>
            </div>
            <div>
              <p class="text-primary/60 text-xs font-medium uppercase tracking-wider mb-0.5">
                Expires
              </p>
              <p class="text-sm text-primary">{formatExp(note.exp)}</p>
            </div>
            <div>
              <p class="text-primary/60 text-xs font-medium uppercase tracking-wider mb-0.5">
                Created
              </p>
              <p class="text-sm text-primary">{formatDate(note.createdAt)}</p>
            </div>
          </div>
        </div>
      {/each}
    </div>
  {/if}
</div>
