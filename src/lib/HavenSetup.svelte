<script>
// @ts-nocheck
    import { LOGO_POSITIONS, logoFromFile, resolvePlaylist } from "#lib/havenSettings.js";

    let { open = false, firstRun = false, settings, onsave, onclose } = $props();

    let dialog;
    let step = $state(0);
    let logo = $state(null);
    let logoPosition = $state("top-center");
    let playlist = $state(null);
    let playlistTitle = $state(null);
    let playlistInput = $state("");
    let error = $state("");
    let busy = $state(false);
    let dragging = $state(false);

    $effect(() => {
        if (!dialog) return;
        if (open && !dialog.open) {
            step = 0;
            logo = settings.logo;
            logoPosition = settings.logoPosition;
            playlist = settings.playlist;
            playlistTitle = settings.playlistTitle;
            playlistInput = "";
            error = "";
            dialog.showModal();
        } else if (!open && dialog.open) {
            dialog.close();
        }
    });

    const showLogo = $derived(!firstRun || step === 0);
    const showPlaylist = $derived(!firstRun || step === 1);

    async function takeFile(file) {
        if (!file) return;
        error = "";
        busy = true;
        try {
            logo = await logoFromFile(file);
        } catch (e) {
            error = e.message;
        } finally {
            busy = false;
        }
    }

    function onDrop(e) {
        e.preventDefault();
        dragging = false;
        takeFile(e.dataTransfer?.files?.[0]);
    }

    async function commitPlaylistInput() {
        if (!playlistInput.trim()) return true;
        busy = true;
        error = "";
        try {
            const found = await resolvePlaylist(playlistInput);
            playlist = found.url;
            playlistTitle = found.title;
            playlistInput = "";
            return true;
        } catch (e) {
            error = e.message;
            return false;
        } finally {
            busy = false;
        }
    }

    async function finish() {
        if (!(await commitPlaylistInput())) return;
        const ok = onsave({ onboarded: true, logo, logoPosition, playlist, playlistTitle });
        if (ok === false) error = "That logo is too big to save in this browser. Try a smaller file.";
    }

    function skipLogo() {
        logo = null;
        error = "";
        step = 1;
    }

    function skipPlaylist() {
        playlist = null;
        playlistTitle = null;
        playlistInput = "";
        finish();
    }

    function onCancel(e) {
        e.preventDefault();
        if (firstRun) onsave({ ...settings, onboarded: true });
        else onclose();
    }
</script>

<dialog bind:this={dialog} class="setup" oncancel={onCancel} aria-labelledby="setup-title">
    <form method="dialog" onsubmit={(e) => { e.preventDefault(); firstRun && step === 0 ? (step = 1) : finish(); }}>
        <header>
            <h2 id="setup-title">{firstRun ? "Set up your Haven" : "Haven settings"}</h2>
            {#if firstRun}
                <p class="lede">lets get your music app set up! first, upload you logo and then add your playlist. if you leave either blank, you can change them in the top right later by clicking the gear icon to open settings :)</p>
                <p class="steps">Step {step + 1} of 2</p>
            {/if}
        </header>

        {#if showLogo}
            <section>
                <h3>Your logo</h3>
                <label
                    class="drop"
                    class:dragging
                    class:filled={logo}
                    ondragover={(e) => { e.preventDefault(); dragging = true; }}
                    ondragleave={() => (dragging = false)}
                    ondrop={onDrop}
                >
                    <input type="file" accept="image/*" onchange={(e) => takeFile(e.currentTarget.files?.[0])} />
                    {#if logo}
                        <img src={logo} alt="Your logo preview" />
                    {:else}
                        <span class="drop-text">Drop your logo here, or <u>choose a file</u></span>
                    {/if}
                </label>
                <div class="row">
                    <p class="hint">something with a transparent background would look the best</p>
                    {#if logo}
                        <button type="button" class="link" onclick={() => (logo = null)}>Remove</button>
                    {/if}
                </div>
                <label class="field">
                    <span>Logo position</span>
                    <select class="text" bind:value={logoPosition}>
                        {#each LOGO_POSITIONS as option (option.value)}
                            <option value={option.value}>{option.label}</option>
                        {/each}
                    </select>
                </label>
            </section>
        {/if}

        {#if showPlaylist}
            <section>
                <h3>Your playlist</h3>
                {#if playlist}
                    <div class="current">
                        <span>Playing <strong>{playlistTitle ?? "your playlist"}</strong></span>
                        <button type="button" class="link" onclick={() => { playlist = null; playlistTitle = null; }}>Use the default</button>
                    </div>
                {/if}
                <input
                    class="text"
                    type="url"
                    inputmode="url"
                    placeholder="https://soundcloud.com/you/sets/your-playlist"
                    aria-label="SoundCloud playlist link"
                    bind:value={playlistInput}
                />
                <p class="hint">
                    {playlist ? "paste a different link to swap it" : "you can paste a public SoundCloud playlist or leave it empty to play the default playlist"}
                </p>
            </section>
        {/if}

        {#if error}
            <p class="error" role="alert">{error}</p>
        {/if}

        <footer>
            {#if firstRun && step === 0}
                <button type="button" class="ghost" onclick={skipLogo} disabled={busy}>Skip</button>
                <button type="submit" class="primary" disabled={busy}>Next</button>
            {:else if firstRun}
                <button type="button" class="ghost back" onclick={() => { step = 0; error = ""; }} disabled={busy}>Back</button>
                <button type="button" class="ghost" onclick={skipPlaylist} disabled={busy}>Skip</button>
                <button type="submit" class="primary" disabled={busy}>{busy ? "Checking…" : "Done"}</button>
            {:else}
                <button type="button" class="ghost" onclick={onclose} disabled={busy}>Cancel</button>
                <button type="submit" class="primary" disabled={busy}>{busy ? "Checking…" : "Save"}</button>
            {/if}
        </footer>
    </form>
</dialog>

<style>
    .setup {
        width: min(460px, calc(100vw - 32px));
        max-height: calc(100dvh - 32px);
        overflow: auto;
        box-sizing: border-box;
        padding: 0;
        border-radius: 1.5rem;
        background: var(--sticker);
        border: 6px solid var(--sticker-edge);
        box-shadow: 0 24px 60px rgb(40 15 0 / 0.35);
    }
    .setup::backdrop {
        background: rgb(40 20 5 / 0.45);
    }
    .setup[open] {
        animation: rise 0.4s cubic-bezier(0.22, 1, 0.36, 1);
    }
    @keyframes rise {
        from { opacity: 0; transform: translateY(16px); }
        to { opacity: 1; transform: translateY(0); }
    }
    .setup,
    .setup * {
        color: var(--sticker-ink);
    }

    form {
        display: flex;
        flex-direction: column;
        gap: 24px;
        padding: 28px 28px 22px;
    }

    h2 {
        margin: 0;
        font-family: var(--font-display);
        font-size: clamp(32px, 4vw, 42px);
        line-height: 0.85;
        letter-spacing: -0.05em;
    }
    .lede {
        margin: 12px 0 0;
        font-size: 15.5px;
        line-height: 1.3;
        color: var(--sticker-ink-soft);
    }
    .steps {
        margin: 10px 0 0;
        font-size: 13.5px;
        color: var(--sticker-ink-soft);
    }

    section {
        display: flex;
        flex-direction: column;
        gap: 10px;
    }
    h3 {
        margin: 0;
        font-family: var(--font-display);
        font-size: 22px;
        line-height: 0.95;
        letter-spacing: -0.03em;
    }

    .drop {
        position: relative;
        display: grid;
        place-items: center;
        min-height: 130px;
        padding: 16px;
        border: 2px dashed oklch(0.34 0.08 45 / 0.35);
        border-radius: 1rem;
        background: var(--sticker-well);
        cursor: pointer;
        transition: border-color 0.15s ease-out, background-color 0.15s ease-out;
    }
    .drop:hover,
    .drop.dragging {
        border-color: var(--rust);
        background: oklch(0.88 0.06 72);
    }
    .drop.filled {
        border-style: solid;
        border-color: transparent;
    }
    .drop:focus-within {
        outline: 3px solid var(--rust-deep);
        outline-offset: 3px;
    }
    .drop input {
        position: absolute;
        inset: 0;
        opacity: 0;
        cursor: pointer;
    }
    .drop img {
        max-width: 100%;
        max-height: 140px;
        object-fit: contain;
        pointer-events: none;
    }
    .drop-text {
        font-size: 16px;
        text-align: center;
        pointer-events: none;
    }

    .row {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;
        gap: 12px;
    }
    .hint {
        margin: 0;
        font-size: 13.5px;
        line-height: 1.3;
        color: var(--sticker-ink-soft);
    }

    .field {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 12px;
        margin-top: 4px;
        font-family: var(--font-display);
        font-size: 17px;
        letter-spacing: -0.02em;
    }

    .current {
        display: flex;
        align-items: baseline;
        justify-content: space-between;
        gap: 12px;
        padding: 10px 14px;
        border-radius: 0.75rem;
        background: var(--sticker-well);
        font-size: 14.5px;
    }
    .current strong {
        font-weight: normal;
        font-family: var(--font-display);
    }

    .text {
        width: 100%;
        box-sizing: border-box;
        padding: 12px 14px;
        border-radius: 0.75rem;
        background: #fff;
        border: 2px solid oklch(0.34 0.08 45 / 0.2);
        color: var(--sticker-ink);
        font-family: var(--font-body);
        font-size: 15px;
    }
    .field select {
        width: auto;
        min-width: 160px;
        padding-right: 36px;
        appearance: none;
        cursor: pointer;
        letter-spacing: normal;
        background:
            url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'><path d='M1 1.5 6 6.5 11 1.5' fill='none' stroke='%23704020' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'/></svg>") no-repeat right 14px center,
            #fff;
    }
    .field select option {
        color: var(--sticker-ink);
    }
    .text::placeholder {
        color: oklch(0.34 0.08 45 / 0.45);
    }
    .text:focus {
        outline: 3px solid var(--rust-deep);
        outline-offset: 2px;
    }

    .error {
        margin: -8px 0 0;
        padding: 10px 14px;
        border-radius: 0.75rem;
        background: var(--rust-deep);
        color: var(--sticker);
        font-size: 14.5px;
    }

    footer {
        display: flex;
        justify-content: flex-end;
        align-items: center;
        gap: 8px;
    }
    footer .back {
        margin-right: auto;
    }

    button {
        border: 0;
        cursor: pointer;
        transition: transform 0.15s ease-out, background-color 0.15s ease-out;
    }
    button:disabled {
        opacity: 0.55;
        cursor: default;
    }
    button:focus-visible {
        outline: 3px solid var(--rust-deep);
        outline-offset: 3px;
    }
    .primary {
        padding: 11px 26px;
        border-radius: 1.25rem;
        background: var(--rust-deep);
        color: var(--sticker);
        font-family: var(--font-display);
        font-size: 20px;
        letter-spacing: -0.03em;
    }
    .primary:hover:not(:disabled) {
        transform: scale(1.04);
    }
    .primary:active:not(:disabled) {
        transform: scale(1);
    }
    .ghost {
        padding: 11px 16px;
        border-radius: 1.25rem;
        background: transparent;
        font-family: var(--font-display);
        font-size: 18px;
        letter-spacing: -0.03em;
    }
    .ghost:hover:not(:disabled) {
        background: oklch(0.6 0.16 48 / 0.12);
    }
    .link {
        flex: none;
        padding: 0;
        background: none;
        font-size: 14px;
        text-decoration: underline;
        text-underline-offset: 3px;
    }
    .link:hover {
        color: var(--rust-deep);
    }
</style>
