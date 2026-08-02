<div class="xcg-time">
    <div class="xcg-time-head">
        <button
            class="xcg-step"
            title="Previous hour"
            on:click={() => select(index - 1)}
            disabled={index <= 0}>‹</button
        >
        <div class="xcg-time-label">{label}</div>
        <button
            class="xcg-step"
            title="Next hour"
            on:click={() => select(index + 1)}
            disabled={index >= profiles.length - 1}>›</button
        >
    </div>

    <div
        class="xcg-scale"
        bind:this={scaleEl}
        on:wheel|nonpassive={onWheel}
        on:keydown={onKey}
        role="listbox"
        aria-label="Forecast time"
        tabindex="0"
    >
        {#each days as day (day.key)}
            <div class="xcg-day">
                <div class="xcg-day-name">{day.label}</div>
                <div class="xcg-hours">
                    {#each day.items as it (it.i)}
                        <button
                            class="xcg-hour"
                            class:sel={it.i === index}
                            class:night={it.night}
                            role="option"
                            aria-selected={it.i === index}
                            title={it.full}
                            bind:this={els[it.i]}
                            on:click={() => select(it.i)}>{it.hour}</button
                        >
                    {/each}
                </div>
            </div>
        {/each}
    </div>
</div>

<script lang="ts">
    import { createEventDispatcher, tick } from 'svelte';
    import type { SoundingProfile } from '../types';

    export let profiles: SoundingProfile[] = [];
    export let index = 0;

    const dispatch = createEventDispatcher();

    let scaleEl: HTMLDivElement;
    let els: HTMLButtonElement[] = [];

    const fmtLabel = new Intl.DateTimeFormat(undefined, {
        weekday: 'short',
        day: 'numeric',
        month: 'short',
        hour: '2-digit',
        minute: '2-digit',
        hour12: false, // always 24-hour clock
    });

    const fmtDay = new Intl.DateTimeFormat(undefined, {
        weekday: 'short',
        day: 'numeric',
        month: 'short',
    });

    const fmtHour = new Intl.DateTimeFormat(undefined, {
        hour: '2-digit',
        minute: '2-digit',
        hour12: false,
    });

    interface Tick {
        i: number;
        hour: string;
        full: string;
        night: boolean;
    }

    interface Day {
        key: string;
        label: string;
        items: Tick[];
    }

    /** Group the hourly profiles into per-day columns of clickable ticks. */
    const group = (ps: SoundingProfile[]): Day[] => {
        const out: Day[] = [];
        ps.forEach((p, i) => {
            const d = new Date(p.ts);
            const key = `${d.getFullYear()}-${d.getMonth()}-${d.getDate()}`;
            if (!out.length || out[out.length - 1].key !== key) {
                out.push({ key, label: fmtDay.format(d), items: [] });
            }
            const h = d.getHours();
            out[out.length - 1].items.push({
                i,
                // Drop ":00" on whole hours to keep the scale compact.
                hour: d.getMinutes() === 0 ? String(h).padStart(2, '0') : fmtHour.format(d),
                full: fmtLabel.format(d),
                night: h < 6 || h >= 21,
            });
        });
        return out;
    };

    $: days = group(profiles);
    $: label = profiles[index] ? fmtLabel.format(new Date(profiles[index].ts)) : '—';

    const select = (i: number) => {
        if (i < 0 || i >= profiles.length || i === index) return;
        index = i;
        dispatch('change', index);
    };

    /** Keep the picked hour visible when it changes from anywhere. */
    const centre = async (i: number) => {
        await tick();
        const el = els[i];
        if (!el || !scaleEl) return;
        const left = el.offsetLeft - scaleEl.clientWidth / 2 + el.offsetWidth / 2;
        scaleEl.scrollTo({ left: Math.max(0, left), behavior: 'smooth' });
    };

    $: if (profiles.length) centre(index);

    /** Vertical wheel scrolls the scale sideways — the strip is one line tall. */
    const onWheel = (e: WheelEvent) => {
        if (!scaleEl) return;
        const d = Math.abs(e.deltaX) > Math.abs(e.deltaY) ? e.deltaX : e.deltaY;
        if (!d || scaleEl.scrollWidth <= scaleEl.clientWidth) return;
        e.preventDefault();
        scaleEl.scrollLeft += d;
    };

    const onKey = (e: KeyboardEvent) => {
        if (e.key === 'ArrowLeft') select(index - 1);
        else if (e.key === 'ArrowRight') select(index + 1);
        else if (e.key === 'Home') select(0);
        else if (e.key === 'End') select(profiles.length - 1);
        else return;
        e.preventDefault();
    };
</script>

<style lang="less">
    .xcg-time {
        margin: 8px 0 4px;
    }
    .xcg-time-head {
        display: flex;
        align-items: center;
        gap: 6px;
        margin-bottom: 4px;
    }
    .xcg-time-label {
        flex: 1;
        text-align: center;
        font-size: 12.5px;
        font-weight: 600;
        color: #e6edf3;
        font-variant-numeric: tabular-nums;
    }
    .xcg-step {
        background: #16212e;
        border: 1px solid #2a3340;
        border-radius: 4px;
        color: #cdd6e0;
        width: 22px;
        height: 22px;
        line-height: 1;
        font-size: 15px;
        cursor: pointer;
    }
    .xcg-step:disabled {
        opacity: 0.35;
        cursor: default;
    }
    .xcg-scale {
        position: relative; // offsetLeft of the ticks is measured against this
        display: flex;
        align-items: stretch;
        gap: 8px;
        overflow-x: auto;
        overflow-y: hidden;
        padding-bottom: 4px;
        scrollbar-width: thin;
        outline: none;
    }
    .xcg-scale::-webkit-scrollbar {
        height: 5px;
    }
    .xcg-scale::-webkit-scrollbar-thumb {
        background: #2a3340;
        border-radius: 3px;
    }
    .xcg-day {
        flex: none;
        border-left: 1px solid #243042;
        padding-left: 6px;
    }
    .xcg-day-name {
        font-size: 10.5px;
        color: #8b97a3;
        white-space: nowrap;
        margin-bottom: 2px;
    }
    .xcg-hours {
        display: flex;
        gap: 2px;
    }
    .xcg-hour {
        flex: none;
        background: #16212e;
        border: 1px solid #223044;
        border-radius: 3px;
        color: #cdd6e0;
        font-size: 11px;
        font-variant-numeric: tabular-nums;
        padding: 3px 5px;
        cursor: pointer;
        white-space: nowrap;
    }
    .xcg-hour.night {
        color: #6f7d8a; // dim the hours outside the soarable window
        background: #111a24;
    }
    .xcg-hour:hover {
        border-color: #4a9eff;
        color: #e6edf3;
    }
    .xcg-hour.sel {
        background: #36d97a;
        border-color: #36d97a;
        color: #0d1b12;
        font-weight: 700;
    }
</style>
