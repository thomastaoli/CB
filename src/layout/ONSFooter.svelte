<script>
	import { themes } from '../config.js';
	import { getContext } from 'svelte';

	export let theme = getContext('theme');
</script>

<footer style="color: {themes[theme]['text']}; background-color: {themes[theme]['pale']};">
	<div class="col-wide" data-analytics="footer">
		<ul>
			<li><a href="https://github.com/thomastaoli/CB" class="link" style="color: {themes[theme]['text']}">Check out the GitHub repo</a></li>
			<li><a href="https://thomastaoli.github.io" class="link" style="color: {themes[theme]['text']}">Go to my homepage</a></li>
		</ul>
		<hr style="border-top-color: {themes[theme]['muted']}" />
		<div class="license">
			Please refer to the
			<a
				href="https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/"
				class="link"
				target="_blank"
				rel="noopener"
				style="color: {themes[theme]['text']}">Open Government Licence v3.0</a> 
			for more information.
		</div>
	</div>
</footer>

<style>
	footer {
		padding-bottom: 72px;
		padding-top: 36px;
		font-size: 18px;
	}
	footer a.link {
		-webkit-box-sizing: border-box;
		box-sizing: border-box;
		text-decoration: underline;
		display: inline-block;
	}
	footer a.link:hover {
		text-decoration: none;
	}
	footer a.link:active {
		outline: 3px solid transparent;
		background-color: #fd0;
		-webkit-box-shadow: 0 -2px #fd0, 0 4px #222;
		box-shadow: 0 -2px #fd0, 0 4px #222;
		color: #222;
		text-decoration: none;
	}
	footer hr {
		margin-bottom: 27px;
		margin-top: 36px;
		border: none;
		border-top: 1px solid #777;
	}
	ul {
		padding: 0;
		margin: 0;
	}
	li {
		margin-right: 18px;
		display: inline-block;
	}
	.logo-img {
		margin-bottom: 27px;
	}
	.ogl-img {
		margin: 0 0.5rem 0.2rem 0;
		width: 59px;
		height: 24px;
		vertical-align: middle;
	}
	.license {
		vertical-align: top;
	}
	a img:hover {
		cursor: pointer;
	}
</style>