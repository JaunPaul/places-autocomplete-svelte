<script lang="ts">
	import { onMount } from 'svelte';
	import * as GMaps from '@googlemaps/js-api-loader';
	const { Loader } = GMaps;

	// Props Interface
	interface Props {
		PUBLIC_GOOGLE_MAPS_API_KEY: string;
		fetchFields?: string[];
		placeholder?: string;
		language?: string;
		region?: string;
		autocomplete?: AutoFill;
		onResponse: (e: Event) => void;
		onError: (error: string) => void;
	}
	// Bindable Props
	let {
		PUBLIC_GOOGLE_MAPS_API_KEY = $bindable(''),
		/**
		 * By default using SKU: Place Detals (Location Only) - 0.005 USD per each
		 * @see https://developers.google.com/maps/documentation/javascript/usage-and-billing#location-placedetails
		 */
		fetchFields = $bindable(['formattedAddress', 'addressComponents']),
		placeholder = $bindable('Search...'),
		language = $bindable('en-GB'),
		region = $bindable('GB'),
		autocomplete = $bindable('off'),
		onResponse = $bindable((e: Event) => {}),
		onError = $bindable((error: string) => {})
	}: Props = $props();

	// Local variables
	let inputRef: HTMLInputElement;
	let currentSuggestion = $state(-1);
	let title: string = $state('');
	let results: any[] = $state([]);
	let token;
	let loader: GMaps.Loader;
	let placesApi: { [key: string]: any } = {};
	//https://developers.google.com/maps/documentation/javascript/reference/autocomplete-data#AutocompleteRequest.includedPrimaryTypes
	let request = $state({
		input: '',
		language: language,
		region: region,
		sessionToken: ''
	});

	$effect(() => {
		if (request.input == '') {
			results = [];
		}
	});

	/**
	 * Reset search input and results.
	 */
	const reset = () => {
		currentSuggestion = -1;
		request.input = '';
		results = [];
		refreshToken(request);
	};

	/**
	 * Make request and get autocomplete suggestions.
	 * @param event
	 */
	const makeAcRequest = async (
		event: Event & { currentTarget: HTMLInputElement }
	): Promise<void> => {
		const target = event.currentTarget as HTMLInputElement;
		if (target?.value == '') {
			title = '';
			request.input = '';
			results = [];
			return;
		}

		request.input = target.value;

		try {
			const { suggestions } =
				await placesApi.AutocompleteSuggestion.fetchAutocompleteSuggestions(request);
			results = [];
			// iterate suggestions and add results to an array
			for (const suggestion of suggestions) {
				// add suggexstions to results
				results.push({
					to_pace: suggestion.placePrediction.toPlace(),
					text: suggestion.placePrediction.text.toString()
				});
			}
		} catch (e: any) {
			onError((e.name || 'An error occurred') + ' - ' + (e.message || 'see console for details.'));
		}
	};
	/**
     * Event handler for clicking on a suggested place.
     //https://developers-dot-devsite-v2-prod.appspot.com/maps/documentation/javascript/reference/autocomplete-data#AutocompleteSuggestion
     * @param place
     */
	const onPlaceSelected = async (place: {
		[x: string]: any;
		fetchFields: (arg0: { fields: string[] }) => any;
		addressComponents: any;
		formattedAddress: string;
	}): Promise<void> => {
		try {
			await place.fetchFields({
				fields: fetchFields
			});
			let placeData = place.toJSON();
			onResponse(placeData);
		} catch (e: any) {
			onError(
				(e.name || 'An error occurred') + ' - ' + (e.message || 'error fetching place details')
			);
		}

		// reset srarch input and results
		reset();
	};

	/**
	 * Helper function to refresh the session token.
	 * @param request
	 */
	const refreshToken = async (request: {
		input?: string;
		language?: string;
		region?: string;
		sessionToken?: any;
	}): Promise<{ input?: string; language?: string; region?: string; sessionToken?: any }> => {
		try {
			token = new placesApi.AutocompleteSessionToken();
			request.sessionToken = token;
			return request;
		} catch (e: any) {
			onError((e.name || 'An error occurred') + ' - ' + (e.message || 'error fetch token'));
			return request;
		}
	};
	/**
	 * Initialize the Google Maps JavaScript API Loader.
	 */
	onMount(async (): Promise<void> => {
		inputRef.focus();

		try {
			loader = new Loader({
				apiKey: PUBLIC_GOOGLE_MAPS_API_KEY,
				version: 'weekly',
				libraries: ['places']
			});

			const { AutocompleteSessionToken, AutocompleteSuggestion } =
				await loader.importLibrary('places');
			placesApi.AutocompleteSessionToken = AutocompleteSessionToken;
			placesApi.AutocompleteSuggestion = AutocompleteSuggestion;
			token = new placesApi.AutocompleteSessionToken();
			request.sessionToken = token;
		} catch (e: any) {
			onError(
				(e.name || 'An error occurred') + ' - ' + (e.message || 'Error loading Google Maps API')
			);
		}
	});
	/**
	 * Handles keyboard events for navigating the suggestions.
	 */
	function onKeyDown(e: KeyboardEvent) {
		if (e.key === 'ArrowDown') {
			currentSuggestion = Math.min(currentSuggestion + 1, results.length - 1);
		} else if (e.key === 'ArrowUp') {
			currentSuggestion = Math.max(currentSuggestion - 1, 0);
		} else if (e.key === 'Enter') {
			e.preventDefault();
			if (currentSuggestion >= 0) {
				onPlaceSelected(results[currentSuggestion].to_pace);
			}
		} else if (e.key === 'Escape') {
			// reset srarch input and results
			reset();
		}
	}
</script>

<svelte:window on:keydown={onKeyDown} />

<div>
	<div class="relative z-50 transform rounded-xl mt-4">
		<div class="pointer-events-none absolute inset-y-0 left-0 flex items-center pl-3">
			<svg
				xmlns="http://www.w3.org/2000/svg"
				class="w-5 h-5"
				viewBox="0 0 24 24"
				fill="none"
				stroke="currentColor"
				stroke-width="2"
				stroke-linecap="round"
				stroke-linejoin="round"><circle cx="11" cy="11" r="8" /><path d="m21 21-4.3-4.3" /></svg
			>
		</div>

		<input
			type="text"
			name="search"
			bind:this={inputRef}
			class="border-1 w-full rounded-md border-0 bg-white px-4 py-2.5 pl-10 pr-10 md:pr-20 text-gray-800 sm:text-sm"
			{placeholder}
			{autocomplete}
			aria-controls="options"
			bind:value={request.input}
			oninput={makeAcRequest}
		/>

		{#if results.length > 0}
			<div class="absolute inset-y-0 right-0 md:flex hidden py-1.5 pr-1.5">
				<kbd
					class="inline-flex items-center rounded border border-gray-300 px-1 font-sans text-xs text-gray-500 w-8 mr-1"
					>Esc</kbd
				>
				<kbd
					class="inline-flex items-center justify-center rounded border border-gray-300 px-1 font-sans text-xs text-gray-500 w-6"
					>&uArr;</kbd
				>
				<kbd
					class="inline-flex items-center rounded border border-gray-400 px-1 font-sans text-xs text-gray-500 justify-center w-6"
					>&dArr;</kbd
				>
			</div>
			<ul
				class="absolute z-50 -mb-2 mt-1 max-h-60 w-full overflow-auto rounded-md bg-white py-1 text-base shadow-lg ring-1 ring-black ring-opacity-5 focus:outline-none sm:text-sm"
				id="options"
			>
				{#each results as place, i}
					<li
						class="z-50 cursor-default select-none py-2 pl-4 text-gray-800 hover:bg-slate-200 hover:text-white"
						class:bg-indigo-500={i === currentSuggestion}
						class:bg-white={i !== currentSuggestion}
						class:text-white={i === currentSuggestion}
						id="option-{i + 1}"
					>
						<!-- svelte-ignore a11y_invalid_attribute -->
						<a
							href="javascript:void(0)"
							class="block w-full"
							tabindex={i + 1}
							onclick={() => onPlaceSelected(place.to_pace)}
						>
							{place.text}
						</a>
					</li>
				{/each}
			</ul>
		{/if}
	</div>
</div>

<style>
	input,
	ul {
		margin: 0;
		padding: 0;
	}
	.absolute,
	.block,
	svg {
		display: block;
	}
	*,
	.border-gray-300 {
		--tw-border-opacity: 1;
	}
	.text-gray-500,
	.text-gray-900,
	.text-white {
		--tw-text-opacity: 1;
	}
	*,
	:after,
	:before {
		box-sizing: border-box;
		border: 0 solid #e5e7eb;
		--tw-border-spacing-x: 0;
		--tw-border-spacing-y: 0;
		--tw-translate-x: 0;
		--tw-translate-y: 0;
		--tw-rotate: 0;
		--tw-skew-x: 0;
		--tw-skew-y: 0;
		--tw-scale-x: 1;
		--tw-scale-y: 1;
		--tw-pan-x: ;
		--tw-pan-y: ;
		--tw-pinch-zoom: ;
		--tw-scroll-snap-strictness: proximity;
		--tw-gradient-from-position: ;
		--tw-gradient-via-position: ;
		--tw-gradient-to-position: ;
		--tw-ordinal: ;
		--tw-slashed-zero: ;
		--tw-numeric-figure: ;
		--tw-numeric-spacing: ;
		--tw-numeric-fraction: ;
		--tw-ring-inset: ;
		--tw-ring-offset-width: 0px;
		--tw-ring-offset-color: #fff;
		--tw-ring-color: rgba(237, 111, 59, 1);
		--tw-ring-offset-shadow: 0 0 #0000;
		--tw-ring-shadow: 0 0 #0000;
		--tw-shadow: 0 0 #0000;
		--tw-shadow-colored: 0 0 #0000;
		--tw-blur: ;
		--tw-brightness: ;
		--tw-contrast: ;
		--tw-grayscale: ;
		--tw-hue-rotate: ;
		--tw-invert: ;
		--tw-saturate: ;
		--tw-sepia: ;
		--tw-drop-shadow: ;
		--tw-backdrop-blur: ;
		--tw-backdrop-brightness: ;
		--tw-backdrop-contrast: ;
		--tw-backdrop-grayscale: ;
		--tw-backdrop-hue-rotate: ;
		--tw-backdrop-invert: ;
		--tw-backdrop-opacity: ;
		--tw-backdrop-saturate: ;
		--tw-backdrop-sepia: ;
		--tw-contain-size: ;
		--tw-contain-layout: ;
		--tw-contain-paint: ;
		--tw-contain-style: ;
	}
	:after,
	:before {
		--tw-content: '';
	}
	:host,
	kbd {
		font-family:
			ui-monospace,
			SFMono-Regular,
			Menlo,
			Monaco,
			Consolas,
			Liberation Mono,
			Courier New,
			monospace;
		font-feature-settings: normal;
		font-variation-settings: normal;
		font-size: 1em;
	}
	input,
	:-moz-focusring {
		outline: auto;
	}
	:-moz-ui-invalid {
		box-shadow: none;
	}
	::-webkit-inner-spin-button,
	::-webkit-outer-spin-button {
		height: auto;
	}
	::-webkit-search-decoration {
		-webkit-appearance: none;
	}
	::-webkit-file-upload-button {
		-webkit-appearance: button;
		font: inherit;
	}
	ul {
		list-style: none;
	}
	.cursor-default,
	:disabled {
		cursor: default;
	}
	svg {
		vertical-align: middle;
	}
	[type='text'],
	input:where(:not([type])),
	select {
		-webkit-appearance: none;
		-moz-appearance: none;
		appearance: none;
		background-color: #fff;
		border-color: #6b7280;
		border-width: 1px;
		border-radius: 0;
		padding: 0.5rem 0.75rem;
		font-size: 1rem;
		line-height: 1.5rem;
		--tw-shadow: 0 0 #0000;
	}
	[type='text']:focus,
	input:where(:not([type])):focus,
	select:focus {
		outline: transparent solid 2px;
		outline-offset: 2px;
		--tw-ring-inset: var(--tw-empty,);
		--tw-ring-offset-width: 0px;
		--tw-ring-offset-color: #fff;
		--tw-ring-color: #ed6f3b;
		--tw-ring-offset-shadow: var(--tw-ring-inset) 0 0 0 var(--tw-ring-offset-width)
			var(--tw-ring-offset-color);
		--tw-ring-shadow: var(--tw-ring-inset) 0 0 0 calc(1px + var(--tw-ring-offset-width))
			var(--tw-ring-color);
		box-shadow: var(--tw-ring-offset-shadow), var(--tw-ring-shadow), var(--tw-shadow);
		border-color: #ed6f3b;
	}
	input::-moz-placeholder {
		color: #6b7280;
		opacity: 1;
	}
	input::placeholder {
		color: #6b7280;
		opacity: 1;
	}

	:root {
		--background: 0 0% 100%;
		--foreground: 222.2 84% 4.9%;
		--muted: 210 40% 96.1%;
		--muted-foreground: 215.4 16.3% 46.9%;
		--popover: 0 0% 100%;
		--popover-foreground: 222.2 84% 4.9%;
		--card: 0 0% 100%;
		--card-foreground: 222.2 84% 4.9%;
		--border: 214.3 31.8% 91.4%;
		--input: 214.3 31.8% 91.4%;
		--primary: 222.2 47.4% 11.2%;
		--primary-foreground: 210 40% 98%;
		--secondary: 210 40% 96.1%;
		--secondary-foreground: 222.2 47.4% 11.2%;
		--accent: 210 40% 96.1%;
		--accent-foreground: 222.2 47.4% 11.2%;
		--destructive: 0 72.2% 50.6%;
		--destructive-foreground: 210 40% 98%;
		--ring: 222.2 84% 4.9%;
		--radius: 0.5rem;
	}
	* {
		border-color: hsl(var(--border) / var(--tw-border-opacity));
	}
	::backdrop {
		--tw-border-spacing-x: 0;
		--tw-border-spacing-y: 0;
		--tw-translate-x: 0;
		--tw-translate-y: 0;
		--tw-rotate: 0;
		--tw-skew-x: 0;
		--tw-skew-y: 0;
		--tw-scale-x: 1;
		--tw-scale-y: 1;
		--tw-pan-x: ;
		--tw-pan-y: ;
		--tw-pinch-zoom: ;
		--tw-scroll-snap-strictness: proximity;
		--tw-gradient-from-position: ;
		--tw-gradient-via-position: ;
		--tw-gradient-to-position: ;
		--tw-ordinal: ;
		--tw-slashed-zero: ;
		--tw-numeric-figure: ;
		--tw-numeric-spacing: ;
		--tw-numeric-fraction: ;
		--tw-ring-inset: ;
		--tw-ring-offset-width: 0px;
		--tw-ring-offset-color: #fff;
		--tw-ring-color: rgb(59 130 246 / 0.5);
		--tw-ring-offset-shadow: 0 0 #0000;
		--tw-ring-shadow: 0 0 #0000;
		--tw-shadow: 0 0 #0000;
		--tw-shadow-colored: 0 0 #0000;
		--tw-blur: ;
		--tw-brightness: ;
		--tw-contrast: ;
		--tw-grayscale: ;
		--tw-hue-rotate: ;
		--tw-invert: ;
		--tw-saturate: ;
		--tw-sepia: ;
		--tw-drop-shadow: ;
		--tw-backdrop-blur: ;
		--tw-backdrop-brightness: ;
		--tw-backdrop-contrast: ;
		--tw-backdrop-grayscale: ;
		--tw-backdrop-hue-rotate: ;
		--tw-backdrop-invert: ;
		--tw-backdrop-opacity: ;
		--tw-backdrop-saturate: ;
		--tw-backdrop-sepia: ;
		--tw-contain-size: ;
		--tw-contain-layout: ;
		--tw-contain-paint: ;
		--tw-contain-style: ;
	}
	.sr-only {
		width: 1px;
		height: 1px;
		padding: 0;
		margin: -1px;
		overflow: hidden;
		clip: rect(0, 0, 0, 0);
		white-space: nowrap;
		border-width: 0;
	}
	.pointer-events-none {
		pointer-events: none;
	}
	.relative {
		position: relative;
	}
	.inset-y-0 {
		top: 0;
		bottom: 0;
	}
	.left-0 {
		left: 0;
	}
	.right-0 {
		right: 0;
	}
	.z-50 {
		z-index: 50;
	}
	.-mb-2 {
		margin-bottom: -0.5rem;
	}
	.mr-1 {
		margin-right: 0.25rem;
	}
	.mt-4 {
		margin-top: 1rem;
	}
	.flex {
		display: flex;
	}
	.inline-flex {
		display: inline-flex;
	}
	.hidden {
		display: none;
	}
	.h-10 {
		height: 2.5rem;
	}
	.h-5 {
		height: 1.25rem;
	}
	.max-h-60 {
		max-height: 15rem;
	}
	.w-5 {
		width: 1.25rem;
	}
	.w-6 {
		width: 1.5rem;
	}
	.w-8 {
		width: 2rem;
	}
	.w-full {
		width: 100%;
	}
	.transform {
		transform: translate(var(--tw-translate-x), var(--tw-translate-y)) rotate(var(--tw-rotate))
			skew(var(--tw-skew-x)) skewY(var(--tw-skew-y)) scaleX(var(--tw-scale-x))
			scaleY(var(--tw-scale-y));
	}
	.select-none {
		-webkit-user-select: none;
		-moz-user-select: none;
		user-select: none;
	}
	.grid-cols-1 {
		grid-template-columns: repeat(1, minmax(0, 1fr));
		grid-template-columns: 1fr;
	}
	.items-center {
		align-items: center;
	}
	.gap-x-4 {
		-moz-column-gap: 1rem;
		column-gap: 1rem;
		row-gap: 1rem;
	}
	.rounded-md {
		border-radius: calc(var(--radius) - 2px);
	}
	.rounded-xl {
		border-radius: 0.75rem;
	}
	.border {
		border-width: 1px;
	}
	.border-gray-300 {
		border-color: rgb(209 213 219 / var(--tw-border-opacity));
	}
	.bg-gray-100 {
		--tw-bg-opacity: 1;
		background-color: rgb(243 244 246 / var(--tw-bg-opacity));
	}
	.bg-indigo-500,
	.hover\:bg-indigo-500:hover {
		--tw-bg-opacity: 1;
		background-color: rgb(99 102 241 / var(--tw-bg-opacity));
	}
	.bg-transparent {
		background-color: transparent;
	}
	.bg-white {
		--tw-bg-opacity: 1;
		background-color: rgb(255 255 255 / var(--tw-bg-opacity));
	}
	.px-1 {
		padding-left: 0.25rem;
		padding-right: 0.25rem;
	}
	.px-4 {
		padding-left: 1rem;
		padding-right: 1rem;
	}
	.py-0 {
		padding-top: 0;
		padding-bottom: 0;
	}
	.py-1 {
		padding-top: 0.25rem;
		padding-bottom: 0.25rem;
	}
	.py-1\.5 {
		padding-top: 0.375rem;
		padding-bottom: 0.375rem;
	}
	.py-2 {
		padding-top: 0.5rem;
		padding-bottom: 0.5rem;
	}
	.py-2\.5 {
		padding-top: 0.625rem;
		padding-bottom: 0.625rem;
	}
	.pl-10 {
		padding-left: 2.5rem;
	}
	.pl-2 {
		padding-left: 0.5rem;
	}
	.pl-3 {
		padding-left: 0.75rem;
	}
	.pl-4 {
		padding-left: 1rem;
	}
	.pr-1\.5 {
		padding-right: 0.375rem;
	}
	.pr-20 {
		padding-right: 5rem;
	}
	.pr-7 {
		padding-right: 1.75rem;
	}
	.text-base {
		font-size: 1rem;
		line-height: 1.5rem;
	}
	.text-sm {
		font-size: 0.875rem;
		line-height: 1.25rem;
	}
	.text-xs {
		font-size: 0.75rem;
		line-height: 1rem;
	}
	.text-gray-500 {
		color: rgb(107 114 128 / var(--tw-text-opacity));
	}
	.text-gray-900 {
		color: rgb(17 24 39 / var(--tw-text-opacity));
	}
	.text-white {
		color: rgb(255 255 255 / var(--tw-text-opacity));
	}
	.shadow-lg {
		--tw-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
		--tw-shadow-colored: 0 10px 15px -3px var(--tw-shadow-color),
			0 4px 6px -4px var(--tw-shadow-color);
		box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000),
			var(--tw-shadow);
	}
	.focus\:ring-2:focus,
	.ring-1 {
		--tw-ring-offset-shadow: var(--tw-ring-inset) 0 0 0 var(--tw-ring-offset-width)
			var(--tw-ring-offset-color);
		box-shadow: var(--tw-ring-offset-shadow), var(--tw-ring-shadow), var(--tw-shadow, 0 0 #0000);
	}
	.ring-1 {
		--tw-ring-shadow: var(--tw-ring-inset) 0 0 0 calc(1px + var(--tw-ring-offset-width))
			var(--tw-ring-color);
	}
	.ring-black {
		--tw-ring-opacity: 1;
		--tw-ring-color: rgb(0 0 0 / var(--tw-ring-opacity));
	}
	.focus\:outline-none:focus {
		outline: transparent solid 2px;
		outline-offset: 2px;
	}
	.focus\:ring-2:focus {
		--tw-ring-shadow: var(--tw-ring-inset) 0 0 0 calc(2px + var(--tw-ring-offset-width))
			var(--tw-ring-color);
	}
	.focus\:ring-inset:focus {
		--tw-ring-inset: inset;
	}
	@media (min-width: 640px) {
		.sm\:text-sm {
			font-size: 0.875rem;
			line-height: 1.25rem;
		}
	}
	@media (min-width: 1024px) {
		.lg\:col-span-2 {
			grid-column: span 2 / span 2;
		}
	}
	@keyframes swipe-out {
		0% {
			transform: translateY(calc(var(--lift) * var(--offset) + var(--swipe-amount)));
			opacity: 1;
		}
		to {
			transform: translateY(
				calc(var(--lift) * var(--offset) + var(--swipe-amount) + var(--lift) * -100%)
			);
			opacity: 0;
		}
	}
	.grid {
		display: grid;
	}
	@media (min-width: 768px) {
		.lg\:grid-cols-6 {
			grid-template-columns: repeat(6, 1fr);
		}
		.lg\:col-span-6 {
			grid-column-start: span 6;
		}
		.lg\:col-span-4 {
			grid-column-start: span 4;
		}
		.lg\:col-span-2 {
			grid-column-start: span 2;
		}
	}
	.my-10 {
		margin-top: 2.5rem;
		margin-bottom: 2.5rem;
	}
	.justify-center {
		justify-content: center;
	}
	.hover\:text-white:hover {
		color: rgb(255 255 255 / var(--tw-text-opacity));
	}
</style>
