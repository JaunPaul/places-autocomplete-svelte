<script>
	// @ts-nocheck
	import PlaceAutocomplete from '$lib/PlaceAutocomplete.svelte';
	import { browser } from '$app/environment';

	// Full address as string
	let formattedAddress = $state('');
	// Formatted address object
	let formattedAddressObj = $state({
		street_number: '',
		street: '',
		town: '',
		county: '',
		country_iso2: '',
		postcode: ''
	});

	/**
	 * @type {never[]}
	 * fullResponse - Unformatted response from Google Places API
	 */
	let fullResponse = $state([]);
	// Google Maps API key
	const PUBLIC_GOOGLE_MAPS_API_KEY = 'AIzaSyCFB0-9qay7P0jZZDw93SsTnyLtZDlch7Q';

	// Countries - optional, if not provided defaults to GB
	let countries = $state([
		{ name: 'United Kingdom', region: 'GB' },
		{ name: 'United States', region: 'US' },
		{ name: 'Switzerland', region: 'CH' },
		{ name: 'Greece', region: 'GR' },
		{ name: 'Russia', region: 'RU' },
		{ name: 'Japan', region: 'JP' }
	]);
	// Error message
	let placesError = $state('');
	// Error handler function
	let onError = (error) => {
		placesError = error;
	};

	let onResponse = (response) => {
		console.log(response);
		formattedAddress = response.formattedAddress;
		fullResponse = response;

		formattedAddressObj = {
			street_number: '',
			street: '',
			town: '',
			county: '',
			country_iso2: '',
			postcode: ''
		};

		for (const component of response.addressComponents) {
			switch (component.types[0]) {
				case 'street_number':
				case 'point_of_interest':
				case 'establishment':
					formattedAddressObj.street_number = component.longText;
					break;
				case 'route':
				case 'premise':
					formattedAddressObj.street = component.longText;
					break;
				case 'postal_town':
					formattedAddressObj.town = component.longText;
					break;
				case 'administrative_area_level_2':
					formattedAddressObj.county = component.longText;
					break;
				case 'country':
					formattedAddressObj.country_iso2 = component.shortText;
					break;
				case 'postal_code':
					formattedAddressObj.postcode = component.longText;
					break;
			}
		}
	};

	// Display response in tabs
	const tabs = [
		{ name: 'Response', id: 1 },
		{ name: 'Formatted Resposne', id: 2 }
	];
	let selectedTab = $state(tabs.find((tab) => tab.id === 1).id);
	const placeholder = 'Search...';
	const language = 'cs-CS';
	const region = 'CZ';
</script>

<div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
	{#if placesError}
		<div
			class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative mt-10"
			role="alert"
		>
			<strong class="font-bold">Error!</strong>
			<span class="block sm:inline">{placesError}</span>
		</div>
	{/if}

	<div class="my-2">
		<PlaceAutocomplete
			{onError}
			{onResponse}
			{PUBLIC_GOOGLE_MAPS_API_KEY}
			{placeholder}
			{language}
			{region}
		/>
	</div>
</div>
