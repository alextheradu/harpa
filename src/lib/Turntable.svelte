<script>
// @ts-nocheck
    import { onMount } from "svelte";
    import { resolve } from "$app/paths";
    import { DEFAULT_PLAYLIST } from "#lib/havenSettings.js";

    let { layout = "corner", playlist = DEFAULT_PLAYLIST } = $props();

    let activePlaylist;
    const POLL_MS = 60_000;

    const WIDGET_OPTS = {
        hide_related: true,
        show_comments: false,
        show_user: false,
        show_reposts: false,
        show_teaser: false,
        visual: false,
    };

    const widgetSrc = (autoPlay) =>
        "https://w.soundcloud.com/player/?url=" + encodeURIComponent(activePlaylist) + "&" +
        Object.entries({ ...WIDGET_OPTS, auto_play: autoPlay }).map(([k, v]) => `${k}=${v}`).join("&");

    let iframe;
    let scoutFrame;
    let disc;
    let widget;
    let scout;

    let loadedIds = "";
    let latestIds = [];
    let updatePending = false;
    let hasPlayed = false;
    let currentId = null;
    let seekTarget = null;
    let seekUntil = 0;
    let progressAt = 0;

    const BARS = 10;
    let viz;
    let wave = null;
    let waveUrl = null;

    async function loadWave(url) {
        waveUrl = url;
        wave = null;
        try {
            const r = await fetch(url);
            const { samples } = await r.json();
            if (url !== waveUrl || !samples?.length) return;
            const sorted = [...samples].sort((a, b) => a - b);
            const lo = sorted[Math.floor(sorted.length * 0.1)];
            const hi = sorted[sorted.length - 1];
            const span = Math.max(1, hi - lo);
            wave = samples.map((v) => Math.min(1, Math.max(0, (v - lo) / span)));
        } catch {
            wave = null;
        }
    }

    let ready = $state(false);
    let playing = $state(false);
    let failed = $state(false);
    let title = $state("tuning in…");
    let artist = $state("");
    let art = $state(null);
    let position = $state(0);
    let duration = $state(0);
    let trackIndex = $state(0);
    let trackCount = $state(0);

    let progress = $derived(duration ? Math.min(1, position / duration) : 0);

    const hiRes = (url) => url?.replace("-large.", "-t500x500.") ?? null;
    const idsOf = (sounds) => (sounds ?? []).map((s) => s.id);
    const wrap = (i, n) => ((i % n) + n) % n;

    function fmt(ms) {
        const s = Math.max(0, Math.floor(ms / 1000));
        return `${Math.floor(s / 60)}:${String(s % 60).padStart(2, "0")}`;
    }

    function loadScript() {
        if (window.SC?.Widget) return Promise.resolve();
        return new Promise((res, rej) => {
            const s = document.createElement("script");
            s.src = "https://w.soundcloud.com/player/api.js";
            s.onload = res;
            s.onerror = rej;
            document.head.appendChild(s);
        });
    }

    const ask = (w, method) => new Promise((res) => w[method](res));

    async function refreshSound() {
        const [s, i, sounds] = await Promise.all([
            ask(widget, "getCurrentSound"),
            ask(widget, "getCurrentSoundIndex"),
            ask(widget, "getSounds"),
        ]);
        trackCount = sounds?.length ?? 0;
        trackIndex = i ?? 0;
        if (!s) return;
        if (s.id !== currentId) {
            seekTarget = null;
            position = 0;
        }
        currentId = s.id;
        const nextWave = s.waveform_url?.replace(/\.png$/, ".json");
        if (nextWave && nextWave !== waveUrl) loadWave(nextWave);
        if (!nextWave) { wave = null; waveUrl = null; }
        title = s.title;
        artist = s.user?.username ?? "";
        art = hiRes(s.artwork_url || s.user?.avatar_url);
        duration = s.duration;
    }

    function reloadPlaylist(startAt, autoplay) {
        updatePending = false;
        seekTarget = null;
        position = 0;
        widget.load(activePlaylist, {
            ...WIDGET_OPTS,
            auto_play: false,
            callback: async () => {
                const sounds = await ask(widget, "getSounds");
                loadedIds = idsOf(sounds).join(",");
                if (autoplay && sounds?.length) widget.skip(wrap(startAt, sounds.length));
                refreshSound();
            },
        });
    }

    function applyUpdate(step) {
        const at = latestIds.indexOf(currentId);
        const base = at >= 0 ? at + step : trackIndex + Math.min(step, 0);
        reloadPlaylist(base, true);
    }

    function checkPlaylist() {
        if (!scout || document.hidden) return;
        scout.load(activePlaylist, {
            ...WIDGET_OPTS,
            auto_play: false,
            callback: async () => {
                const ids = idsOf(await ask(scout, "getSounds"));
                if (!ids.length || ids.join(",") === loadedIds) return;
                latestIds = ids;
                if (hasPlayed) updatePending = true;
                else reloadPlaylist(0, false);
            },
        });
    }

    let keepAlive;

    // chrome requires at least five seconds of media for full audio focus
    function silentWavUrl(seconds = 6, rate = 8000) {
        const n = seconds * rate;
        const buf = new ArrayBuffer(44 + n);
        const v = new DataView(buf);
        const str = (o, t) => [...t].forEach((c, i) => v.setUint8(o + i, c.charCodeAt(0)));
        str(0, "RIFF"); v.setUint32(4, 36 + n, true); str(8, "WAVE");
        str(12, "fmt "); v.setUint32(16, 16, true); v.setUint16(20, 1, true); v.setUint16(22, 1, true);
        v.setUint32(24, rate, true); v.setUint32(28, rate, true); v.setUint16(32, 1, true); v.setUint16(34, 8, true);
        str(36, "data"); v.setUint32(40, n, true);
        new Uint8Array(buf, 44).fill(128);
        return URL.createObjectURL(new Blob([buf], { type: "audio/wav" }));
    }

    function syncKeepAlive() {
        if (!keepAlive) return;
        if (playing) keepAlive.play().catch(() => {});
        else keepAlive.pause();
    }

    $effect(() => {
        playing;
        syncKeepAlive();
    });

    function play() {
        if (!ready) return;
        keepAlive?.play().catch(() => {});
        widget.play();
    }

    function pause() {
        if (!ready) return;
        keepAlive?.pause();
        widget.pause();
    }

    $effect(() => {
        const wanted = playlist || DEFAULT_PLAYLIST;
        if (!ready || wanted === activePlaylist) return;
        activePlaylist = wanted;
        loadedIds = "";
        latestIds = [];
        reloadPlaylist(0, playing);
    });

    function toggle() {
        if (playing) pause();
        else play();
    }

    async function next() {
        if (!ready) return;
        if (updatePending) return applyUpdate(1);
        const [i, sounds] = await Promise.all([ask(widget, "getCurrentSoundIndex"), ask(widget, "getSounds")]);
        if (i < (sounds?.length ?? 1) - 1) widget.next();
        else widget.skip(0);
    }

    async function prev() {
        if (!ready) return;
        if (position > 3000) return seek(0);
        if (updatePending) return applyUpdate(-1);
        const [i, sounds] = await Promise.all([ask(widget, "getCurrentSoundIndex"), ask(widget, "getSounds")]);
        if (i > 0) widget.prev();
        else widget.skip((sounds?.length ?? 1) - 1);
    }

    function seek(ms) {
        if (!ready || !Number.isFinite(ms)) return;
        const target = Math.max(0, Math.min(duration || ms, ms));
        position = target;
        seekTarget = target;
        seekUntil = performance.now() + 4000;
        widget.seekTo(target);
    }

    function onProgress(ms) {
        if (seekTarget !== null) {
            const stale = Math.abs(ms - seekTarget) > 1500 && performance.now() < seekUntil;
            if (stale) return;
            seekTarget = null;
        }
        position = ms;
        progressAt = performance.now();
    }

    function seekFromPointer(e) {
        if (!ready || !duration) return;
        const r = e.currentTarget.getBoundingClientRect();
        seek(((e.clientX - r.left) / r.width) * duration);
    }

    function setupMediaKeys() {
        if (!("mediaSession" in navigator)) return () => {};
        const session = navigator.mediaSession;
        const handlers = {
            play,
            pause,
            stop: () => { pause(); seek(0); },
            nexttrack: next,
            previoustrack: prev,
            seekto: (d) => seek(d.seekTime * 1000),
            seekbackward: (d) => seek(position - (d.seekOffset ?? 10) * 1000),
            seekforward: (d) => seek(position + (d.seekOffset ?? 10) * 1000),
        };
        const registered = [];
        for (const [action, handler] of Object.entries(handlers)) {
            try {
                session.setActionHandler(action, handler);
                registered.push(action);
            } catch {
                // nothing HAHA catch nothing or wait maybe it catches 21? get it?? catch 21??
            }
        }
        return () => {
            for (const action of registered) session.setActionHandler(action, null);
            session.metadata = null;
            session.playbackState = "none";
            session.setPositionState?.();
        };
    }

    $effect(() => {
        if (!("mediaSession" in navigator) || !("MediaMetadata" in window) || !ready) return;
        navigator.mediaSession.metadata = new MediaMetadata({
            title,
            artist,
            album: "Haven NYC",
            artwork: art ? [{ src: art, sizes: "500x500", type: "image/jpeg" }] : [],
        });
    });

    $effect(() => {
        if (!("mediaSession" in navigator)) return;
        navigator.mediaSession.playbackState = playing ? "playing" : ready ? "paused" : "none";
    });

    $effect(() => {
        if (!("mediaSession" in navigator) || !navigator.mediaSession.setPositionState) return;
        if (!ready || !Number.isFinite(duration) || duration <= 0) {
            navigator.mediaSession.setPositionState();
            return;
        }
        try {
            navigator.mediaSession.setPositionState({
                duration: duration / 1000,
                position: Math.max(0, Math.min(position, duration)) / 1000,
                playbackRate: 1,
            });
        } catch {}
    });

    function seekFromKey(e) {
        if (!ready) return;
        const step = { ArrowLeft: -5000, ArrowRight: 5000 }[e.key];
        if (!step) return;
        e.preventDefault();
        seek(position + step);
    }

    onMount(() => {
        let raf;

        keepAlive = new Audio(silentWavUrl());
        keepAlive.loop = true;
        const cleanupMediaKeys = setupMediaKeys();
        let poll;
        let cancelled = false;

        const reduced = matchMedia("(prefers-reduced-motion: reduce)").matches;
        const SPIN_DEG_PER_MS = reduced ? 0.012 : 0.054;
        let angle = 0, vel = 0, last = performance.now();
        const levels = new Array(BARS).fill(0.08);

        const loudnessAt = (ms) => {
            if (!wave || !duration) return 0.6;
            const i = Math.floor((ms / duration) * wave.length);
            return wave[Math.min(wave.length - 1, Math.max(0, i))];
        };

        const drawBars = (t, dt) => {
            if (!viz) return;
            const now = playing ? position + Math.min(1000, performance.now() - progressAt) : position;
            const step = duration ? duration / (wave?.length || 1800) : 100;
            for (let i = 0; i < BARS; i++) {
                const v = loudnessAt(now + (i - BARS / 2) * step * 2);
                const wobble =
                    (0.5 + 0.5 * Math.sin(t * 0.0042 * (1 + i * 0.37) + i * 1.7)) *
                    (0.6 + 0.4 * Math.sin(t * 0.0023 * (1 + i * 0.21) + i));
                const target = playing && !reduced ? 0.12 + 0.88 * v * (0.35 + 0.65 * wobble) : 0.08;
                levels[i] += (target - levels[i]) * (1 - Math.exp(-dt / 110));
                viz.children[i].style.transform = `scaleY(${levels[i].toFixed(3)})`;
            }
        };
        const spin = (t) => {
            const dt = Math.min(64, t - last);
            last = t;
            const target = playing ? SPIN_DEG_PER_MS : 0;
            vel += (target - vel) * (1 - Math.exp(-dt / 450));
            angle = (angle + vel * dt) % 360;
            if (disc) disc.style.transform = `rotate(${angle}deg)`;
            drawBars(t, dt);
            raf = requestAnimationFrame(spin);
        };
        raf = requestAnimationFrame(spin);

        const onVisible = () => { if (!document.hidden) checkPlaylist(); };

        loadScript().then(() => {
            if (cancelled) return;
            const E = window.SC.Widget.Events;

            activePlaylist = playlist || DEFAULT_PLAYLIST;
            iframe.src = widgetSrc(true);
            widget = window.SC.Widget(iframe);
            widget.bind(E.READY, async () => {
                ready = true;
                loadedIds = idsOf(await ask(widget, "getSounds")).join(",");
                refreshSound();
            });
            widget.bind(E.PLAY, () => { playing = true; hasPlayed = true; refreshSound(); });
            widget.bind(E.PAUSE, () => { playing = false; });
            widget.bind(E.PLAY_PROGRESS, (e) => onProgress(e.currentPosition));
            widget.bind(E.SEEK, (e) => onProgress(e.currentPosition));
            widget.bind(E.FINISH, async () => {
                playing = false;
                if (updatePending) return applyUpdate(1);
                const [i, sounds] = await Promise.all([ask(widget, "getCurrentSoundIndex"), ask(widget, "getSounds")]);
                if (i >= (sounds?.length ?? 1) - 1) widget.skip(0);
            });

            scoutFrame.src = widgetSrc(false);
            scout = window.SC.Widget(scoutFrame);
            poll = setInterval(checkPlaylist, POLL_MS);
            document.addEventListener("visibilitychange", onVisible);
        }).catch(() => {
            failed = true;
            title = "the radio's quiet";
            artist = "couldn't reach soundcloud";
        });

        return () => {
            cancelled = true;
            cancelAnimationFrame(raf);
            cleanupMediaKeys();
            keepAlive.pause();
            URL.revokeObjectURL(keepAlive.src);
            keepAlive = null;
            clearInterval(poll);
            document.removeEventListener("visibilitychange", onVisible);
        };
    });
</script>

<div class="deck {layout}" class:playing>
    <iframe
        bind:this={iframe}
        class="sc"
        title="SoundCloud audio"
        aria-hidden="true"
        tabindex="-1"
        allow="autoplay; encrypted-media"
    ></iframe>
    <iframe
        bind:this={scoutFrame}
        class="sc"
        title="SoundCloud playlist check"
        aria-hidden="true"
        tabindex="-1"
    ></iframe>

    <div class="slot">
        <div class="vinyl" bind:this={disc}>
            <div class="label">
                {#if art}
                    <img src={art} alt="Cover art for {title}" />
                {:else}
                    <span>haven</span>
                {/if}
            </div>
            <div class="hole"></div>
        </div>
        <div class="sheen"></div>
        <div class="notes" aria-hidden="true">
            <span>♪</span><span>♫</span>
        </div>
    </div>

    <div class="tagwrap">
        <img class="pudding" src={resolve("haven/pudding.webp")} alt="Pudding the hedgehog" />
        <div class="viz" bind:this={viz} aria-hidden="true">
            {#each { length: BARS } as _}<span></span>{/each}
        </div>
        <div class="tag">
            <div aria-live="polite">
                <p class="title" title={title}>{title}</p>
                <p class="artist">{artist || " "}</p>
            </div>

            <div
                class="progress"
                role="slider"
                tabindex="0"
                aria-label="Seek"
                aria-valuemin="0"
                aria-valuemax={Math.round(duration / 1000)}
                aria-valuenow={Math.round(position / 1000)}
                aria-valuetext="{fmt(position)} of {fmt(duration)}"
                onpointerdown={seekFromPointer}
                onkeydown={seekFromKey}
            >
                <div class="fill" style:width="{progress * 100}%"></div>
            </div>

            <div class="foot">
                <span class="meta">
                    {#if failed}off air{:else}{fmt(position)} / {fmt(duration)}{#if trackCount}<span class="no">no. {trackIndex + 1} of {trackCount}</span>{/if}{/if}
                </span>
                <div class="controls">
                    <button aria-label="Previous" title="Previous" onclick={prev} disabled={!ready}>
                        <svg viewBox="0 0 24 24" aria-hidden="true"><path class="s" d="M6 5.5v13" /><path d="M18 6.5 9 12l9 5.5z" /></svg>
                    </button>
                    <button class="play" aria-label={playing ? "Pause" : "Play"} title={playing ? "Pause" : "Play"} onclick={toggle} disabled={!ready}>
                        {#if playing}
                            <svg viewBox="0 0 24 24" aria-hidden="true"><path class="s thick" d="M9 7v10M15 7v10" /></svg>
                        {:else}
                            <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M9 6.5 18 12l-9 5.5z" /></svg>
                        {/if}
                    </button>
                    <button aria-label="Next" title="Next" onclick={next} disabled={!ready}>
                        <svg viewBox="0 0 24 24" aria-hidden="true"><path class="s" d="M18 5.5v13" /><path d="M6 6.5 15 12l-9 5.5z" /></svg>
                    </button>
                </div>
            </div>
        </div>
    </div>
</div>

<style>
    .deck {
        --disc: 112px;
        --tag-w: 236px;

        position: fixed;
        z-index: 1000;
        width: calc(var(--tag-w) + 84px);
        height: 124px;
    }
    .deck.corner {
        left: 22px;
        bottom: 20px;
        transform-origin: bottom left;
    }
    .deck.center {
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%) scale(1.5);
    }
    .deck.bottom {
        left: 50%;
        bottom: 28px;
        transform: translateX(-50%) scale(1.15);
        transform-origin: bottom center;
    }
    @media (max-width: 600px) {
        .deck.corner { left: 12px; bottom: 12px; transform: scale(0.85); }
        .deck.center { transform: translate(-50%, -50%) scale(1.1); }
        .deck.bottom { bottom: 16px; transform: translateX(-50%) scale(0.9); }
    }

    .sc {
        position: absolute;
        z-index: -1;
        left: 0;
        bottom: 0;
        width: 300px;
        height: 166px;
        opacity: 0.01;
        pointer-events: none;
        border: 0;
    }

    .viz {
        position: absolute;
        z-index: 0;
        right: 16px;
        bottom: calc(100% - 6px);
        display: flex;
        align-items: flex-end;
        gap: 3px;
        height: 30px;
    }
    .viz span {
        width: 5px;
        height: 100%;
        border-radius: 3px 3px 0 0;
        background: var(--rust);
        transform: scaleY(0.08);
        transform-origin: bottom;
        will-change: transform;
    }

    .pudding {
        position: absolute;
        z-index: 0;
        width: 150px;
        left: 12px;
        bottom: calc(100% - 24px);
        transform-origin: bottom center;
    }
    .playing .pudding {
        animation: bob 1.6s ease-in-out infinite;
    }
    @keyframes bob {
        0%, 100% { transform: translateY(0) rotate(0deg); }
        25% { transform: translateY(-3px) rotate(-3deg); }
        75% { transform: translateY(-3px) rotate(3deg); }
    }

    .slot {
        position: absolute;
        z-index: 1;
        bottom: 6px;
        left: calc(var(--tag-w) - var(--disc) + 10px);
        width: var(--disc);
        height: var(--disc);
        transform: translateX(26px);
        transition: transform 1.1s cubic-bezier(0.22, 1, 0.36, 1);
    }
    .playing .slot {
        transform: translateX(64px);
    }

    .vinyl {
        position: absolute;
        inset: 0;
        border-radius: 50%;
        background: repeating-radial-gradient(circle, #1d1714 0 1.1px, #261e1a 1.1px 2.4px);
        box-shadow:
            0 0 0 4px #fff,
            0 8px 18px rgb(0 0 0 / 0.35),
            inset 0 0 0 3px #120e0c;
        will-change: transform;
    }
    .label {
        position: absolute;
        inset: 24%;
        border-radius: 50%;
        overflow: hidden;
        display: grid;
        place-items: center;
        background: var(--rust);
        box-shadow: 0 0 0 4px #120e0c;
    }
    .label img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }
    .label span {
        font-family: var(--font-display);
        font-size: 12px;
        color: var(--sticker);
    }
    .hole {
        position: absolute;
        left: 50%;
        top: 50%;
        width: 6px;
        height: 6px;
        margin: -3px 0 0 -3px;
        border-radius: 50%;
        background: #fff;
        box-shadow: 0 0 0 1px rgb(0 0 0 / 0.5);
    }
    .sheen {
        position: absolute;
        inset: 0;
        border-radius: 50%;
        pointer-events: none;
        background: conic-gradient(
            from 20deg,
            transparent 0 8%,
            rgb(255 255 255 / 0.12) 13%,
            transparent 20% 52%,
            rgb(255 255 255 / 0.07) 60%,
            transparent 68%
        );
        mix-blend-mode: screen;
    }

    .notes {
        position: absolute;
        right: 6px;
        top: -4px;
        pointer-events: none;
    }
    .notes span {
        position: absolute;
        font-size: 18px;
        font-family: system-ui, sans-serif;
        color: #fff;
        opacity: 0;
        text-shadow: 0 0 8px var(--haven-orange-deep);
    }
    .playing .notes span {
        animation: drift 6s ease-out infinite;
    }
    .playing .notes span:nth-child(2) {
        animation-delay: 3s;
        left: 12px;
        font-size: 15px;
    }
    @keyframes drift {
        0% { opacity: 0; transform: translate(0, 8px) rotate(-10deg); }
        15% { opacity: 1; }
        100% { opacity: 0; transform: translate(16px, -54px) rotate(14deg); }
    }

    .tagwrap {
        position: absolute;
        z-index: 2;
        left: 0;
        bottom: 0;
        width: var(--tag-w);
    }
    .tag {
        position: relative;
        z-index: 1;
        box-sizing: border-box;
        padding: 14px 16px 12px;
        border-radius: 1.25rem;
        background: var(--sticker);
        border: 5px solid var(--sticker-edge);
        box-shadow: 0 6px 18px rgb(60 25 5 / 0.3);
    }
    .tag,
    .tag * {
        color: var(--sticker-ink);
    }

    .title,
    .artist {
        margin: 0;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .title {
        font-family: var(--font-display);
        font-size: 23px;
        line-height: 0.95;
        letter-spacing: -0.03em;
        padding-bottom: 2px;
    }
    .artist {
        font-size: 14px;
        line-height: 1.2;
        color: var(--sticker-ink-soft);
    }

    .progress {
        position: relative;
        height: 5px;
        margin: 12px 0 8px;
        border-radius: 99px;
        background: oklch(0.34 0.08 45 / 0.16);
        cursor: pointer;
        touch-action: none;
    }
    .progress::after {
        content: "";
        position: absolute;
        inset: -9px 0;
    }
    .progress:focus-visible {
        outline: 3px solid var(--rust-deep);
        outline-offset: 4px;
    }
    .fill {
        height: 100%;
        border-radius: inherit;
        background: var(--rust);
        transition: width 0.25s linear;
    }

    .foot {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 8px;
    }
    .meta {
        display: flex;
        flex-direction: column;
        font-size: 12.5px;
        line-height: 1.2;
        color: var(--sticker-ink-soft);
    }
    .meta .no {
        color: var(--sticker-ink-soft);
        font-size: 11.5px;
    }

    .controls {
        display: flex;
        align-items: center;
        gap: 4px;
    }
    .controls button {
        display: grid;
        place-items: center;
        width: 32px;
        height: 32px;
        padding: 0;
        border: 0;
        border-radius: 50%;
        background: transparent;
        cursor: pointer;
        transition: transform 0.15s ease-out;
    }
    .controls button.play {
        width: 42px;
        height: 42px;
        background: var(--rust-deep);
    }
    .controls button.play,
    .controls button.play * {
        color: var(--sticker);
    }
    .controls svg {
        width: 20px;
        height: 20px;
        fill: currentColor;
        stroke: currentColor;
        stroke-width: 2.2;
        stroke-linejoin: round;
        stroke-linecap: round;
    }
    .controls svg .s {
        fill: none;
        stroke-width: 2.6;
    }
    .controls svg .thick {
        stroke-width: 3.6;
    }
    .controls button:hover:not(:disabled) {
        transform: scale(1.08);
    }
    .controls button:active:not(:disabled) {
        transform: scale(1);
    }
    .controls button:focus-visible {
        outline: 3px solid var(--rust-deep);
        outline-offset: 2px;
    }
    .controls button:disabled {
        opacity: 0.5;
        cursor: default;
    }

    @media (prefers-reduced-motion: reduce) {
        .playing .pudding,
        .playing .notes span { animation: none; }
        .slot { transition: none; }
    }
</style>
