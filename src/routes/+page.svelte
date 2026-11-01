<script>
    import { onMount } from "svelte";
    import { fade } from "svelte/transition";

    let width = $state(0);
    let height = $state(0);

    let player = $state(0);
    let playerCache = $state(0);

    function muteToggle() {
        if (player != -1) {
            playerCache = player;
        }
        
        player = (player == -1 ? playerCache : -1);
    }

    let max = 5;

    /**
     * @param {number} mode
     */
    function cycleTrack (mode) {
        if (mode == 1) {
            if (player == max) {
                player = 0;
            }
            else {
                player++;
            }
        }
        else {
            if (player == 0) {
                player = max;
            }
            else {
                player--;
            }
        }
    }



    function fullscreen() {
        if (document.fullscreenElement) {
            return;
        }
        // @ts-ignore
        document.getElementById("view").requestFullscreen();
    }
    onMount(() => {
        window.addEventListener("fullscreenchange", (e) => {
            if (!document.fullscreenElement) {
                fullscreenEnabled = false;
                return;
            }
            fullscreenEnabled = true;
        })
    })

    let fullscreenEnabled = $state(false);
</script>

<style>
    #background {
        position: fixed;
        top: 0;
        transform: translate(-50%, 0%);
        left: 50%;
        z-index: 0;
        user-select: none;
        -webkit-user-drag: none;
        -moz-user-drag: none;
    }
    #treeLeft {
        position: fixed;
        z-index: 1;
        left: 0;
        height: 100%;
        user-select: none;
        -webkit-user-drag: none;
        -moz-user-drag: none;
        animation: treeLeft 15s infinite ease-in-out;
    }

    @keyframes treeLeft {
        0% {
            transform: translateX(0px)
        }
        50% {
            transform: translateX(35px)
        }
        50% {
            transform: translateX(-35px)
        }
    }

    #treeRight {
        position: fixed;
        z-index: 1;
        right: 0;
        height: 100%;
        user-select: none;
        -webkit-user-drag: none;
        -moz-user-drag: none;
        animation: treeRight 15s infinite ease-in-out;
    }

    @keyframes treeRight {
        0% {
            transform: translateX(0px)
        }
        50% {
            transform: translateX(-20px)
        }
        50% {
            transform: translateX(20px)
        }
    }


    #player {
        z-index: 1000;
        position: fixed;
        left: 20px;
        bottom: 20px;

        #container {
            position: relative;
        }
        #playerFrame {
            position: static;
            z-index: 1000;
        }
        #pudding {
            width: 180px;
            position: static;
            z-index: 999;
            transform: translateY(5px);
        }
        #buttons {
            position: absolute;
            transform: translateY(-90px) translateX(190px);
            button {
                margin: 5px;
            }
        }
    }
    #player.mute {
        transform: translateY(19px);
    }

    button {
        background-color: rgba(198, 110, 10, 0.561);
        border: solid rgb(221, 158, 80) 4.5px;
        border-radius: 20px;
        padding: 10px;
        color: white;
        span {
            font-size: 40px;
            transform: translateY(2px);
            user-select: none;
        }
        transition: background-color 0.1s ease-in;
    }
    button:hover {
        background-color: rgb(198, 110, 10);
        cursor: pointer;
    }
    button:active {
        transform: scale(1.1);
        border: solid rgb(216, 168, 110) 4.5px;
    }
    button.mute {
        background-color: rgb(198, 110, 10);
        transform: scale(1.1);
        border: solid rgb(216, 168, 110) 4.5px;
    }

    #fullscreen {
        position: fixed;
        z-index: 1000;
        top: 15px;
        right: 20px;
        transform: scale(0.8);
    }
    #fullscreen.dissapear {
        display: none;
    }

</style>

<svelte:window bind:innerWidth={width} bind:innerHeight={height}></svelte:window>
<div id="view">
    <img id="background" src="/haven/background.webp" alt="Forest background" style:width={((width >= 990) ? 100 : 180)}%/>
    <img id="treeLeft" src="/haven/treeLeft.webp" alt="Tree" />
    <img id="treeRight" src="/haven/treeRight.webp" alt="Tree" />

    <div id="player" class:mute={player == -1}>
        <div id="container">
            <img id="pudding" src="/haven/pudding.webp" alt="Hedgehog" />
            <div id="buttons">
                <button title="Toggle Music" onclick={muteToggle} class:mute={player == -1}><span translate="no" class="material-symbols-outlined">music_off</span></button>
                {#if player != -1}
                <button title="Previous Track" onclick={() => {cycleTrack(-1)}} transition:fade><span translate="no" class="material-symbols-outlined">skip_previous</span></button>
                <button title="Next Track" onclick={() => {cycleTrack(1)}} transition:fade><span translate="no" class="material-symbols-outlined">skip_next</span></button>
                {/if}

            </div>
            {#if player == 0}
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/soundcloud%253Atracks%253A859871056&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Darumadrop One,Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {:else if player == 1}
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/soundcloud%253Atracks%253A966761764&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {:else if player == 2}
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="300" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/playlists/soundcloud%253Aplaylists%253A468765630&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {:else if player == 3}
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="300" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/playlists/soundcloud%253Aplaylists%253A1746423624&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {:else if player == 4}
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/soundcloud%253Atracks%253A369089762&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {:else if player == 5} 
            <iframe id="playerFrame" title="Music Player" style:font-family="Darumadrop One" width="100%" height="300" scrolling="no" frameborder="no" allow="autoplay; encrypted-media" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/playlists/soundcloud%253Aplaylists%253A1353215899&color=%23ff5500&auto_play=true&hide_related=true&show_comments=false&show_user=false&show_reposts=false&show_teaser=false"></iframe><div style="font-size: 10px; color: #cccccc;line-break: anywhere;word-break: normal;overflow: hidden;white-space: nowrap;text-overflow: ellipsis; font-family: Interstate,Lucida Grande,Lucida Sans Unicode,Lucida Sans,Garuda,Verdana,Tahoma,sans-serif;font-weight: 100;"></div>
            {/if}
        </div>
    </div>
    <div id="fullscreen" class:dissapear={fullscreenEnabled}>
        <button title="Enter Fullscreen" onclick={fullscreen} translate="no"><span class="material-symbols-outlined">fullscreen</span></button>
        <!--<p>Use {"<f>"} key to enter fullscreen</p>-->
    </div>
</div>