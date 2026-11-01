<script>
	import favicon from '#lib/assets/favicon.svg';
	import "#lib/styles.css";

	let { children } = $props();
</script>

<svelte:head>
	<title>Harpa | Haven</title>
	<link rel="icon" href={favicon} />
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com">
	<link href="https://fonts.googleapis.com/css2?family=Darumadrop+One&display=swap" rel="stylesheet">
	<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@24,400,0,0&icon_names=fullscreen,music_off,skip_next,skip_previous" />
</svelte:head>

{@render children()}
