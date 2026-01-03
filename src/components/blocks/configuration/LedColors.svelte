<script>
	import { onMount } from "svelte";
	import { _ } from 'svelte-i18n'
	import Borders from "./../../ui/Borders.svelte";
	import Box from "./../../ui/Box.svelte";
	import { config_store } from "./../../../lib/stores/config.js";
	import { serialQueue } from "./../../../lib/queue.js"

	let mounted = false

	// LED color states matching OpenEVSE LCD colors
	// Actual EVSE state-to-color mapping: NOT_CONNECTED->GREEN, CONNECTED->YELLOW, CHARGING->TEAL
	const ledStates = [
		{ key: 'led_color_off',    name: 'Off',    desc: 'LED off state (black)', default: '000000' },
		{ key: 'led_color_red',    name: 'Red',    desc: 'Error/Fault states ', default: 'FF6347' },
		{ key: 'led_color_green',  name: 'Green',  desc: 'Not Connected / Waiting for vehicle', default: '32CD32' },
		{ key: 'led_color_yellow', name: 'Yellow', desc: 'Connected / Ready to charge', default: 'FFD700' },
		{ key: 'led_color_blue',   name: 'Blue',   desc: 'Unused (reserved)', default: '1E90FF' },
		{ key: 'led_color_violet', name: 'Violet', desc: 'Sleeping/Disabled (vehicle not connected)', default: 'BA55D3' },
		{ key: 'led_color_teal',   name: 'Teal',   desc: 'Charging OR Sleeping/Disabled (vehicle connected)', default: '48D1CC' },
		{ key: 'led_color_white',  name: 'White',  desc: 'Default/Fallback state', default: 'FFFFFF' }
	]

	let formdata = {}

	// Initialize form data
	ledStates.forEach(state => {
		formdata[state.key] = {
			val: state.default,
			status: "",
			input: undefined,
			req: false
		}
	})

	// Convert 24-bit integer to hex string
	function intToHex(value) {
		if (typeof value === 'number') {
			return value.toString(16).padStart(6, '0').toUpperCase()
		}
		return value || '000000'
	}

	// Convert hex string to 24-bit integer
	function hexToInt(hex) {
		// Remove any # prefix and ensure it's a valid hex string
		const cleanHex = hex.replace(/^#/, '').replace(/[^0-9A-Fa-f]/g, '')
		return parseInt(cleanHex || '0', 16)
	}

	let updateFormData = () => {
		ledStates.forEach(state => {
			if ($config_store[state.key] !== undefined) {
				const configValue = $config_store[state.key]
				// Handle both integer and string values from config_store
				formdata[state.key].val = typeof configValue === 'number' 
					? intToHex(configValue)
					: configValue
			}
		})
	}

	let setProperty = async (prop) => {
		// Convert hex string to integer before submitting
		const hexValue = formdata[prop].val
		const intValue = hexToInt(hexValue)
		
		console.log(`Setting ${prop}: hex=${hexValue}, int=${intValue}`)
		
		// Mark as loading
		formdata[prop].status = "loading"
		
		// Submit directly to config_store
		const data = {}
		data[prop] = intValue
		
		if (await serialQueue.add(() => config_store.upload(data))) {
			formdata[prop].status = "ok"
			// Update config_store to reflect the new value
			config_store.update(current => ({...current, [prop]: intValue}))
			// Clear status after a short delay
			setTimeout(() => {
				formdata[prop].status = ""
			}, 2000)
		} else {
			formdata[prop].status = "error"
		}
	}

	let testColor = async (prop) => {
		// Convert hex string to integer and send to test endpoint
		const hexValue = formdata[prop].val
		const intValue = hexToInt(hexValue)
		
		console.log(`Testing LED color ${prop}: hex=${hexValue}, int=${intValue}`)
		
		// Mark as testing
		formdata[prop].status = "testing"
		
		try {
			const response = await fetch('/testled', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
				},
				body: JSON.stringify({ color: intValue })
			})
			
			if (response.ok) {
				formdata[prop].status = "ok"
				// Clear status after 3 seconds (same as LED test duration)
				setTimeout(() => {
					formdata[prop].status = ""
				}, 3000)
			} else {
				formdata[prop].status = "error"
				setTimeout(() => {
					formdata[prop].status = ""
				}, 2000)
			}
		} catch (error) {
			console.error('LED test failed:', error)
			formdata[prop].status = "error"
			setTimeout(() => {
				formdata[prop].status = ""
			}, 2000)
		}
	}

	let resetToDefaults = async () => {
		for (const state of ledStates) {
			formdata[state.key].val = state.default
			await setProperty(state.key)
		}
	}

	onMount(() => {
		updateFormData()
		mounted = true
	})
</script>

<style>
	.color-input-wrapper {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.color-preview {
		width: 50px;
		height: 50px;
		border-radius: 8px;
		border: 2px solid #dbdbdb;
		cursor: pointer;
		transition: transform 0.2s;
	}

	.color-preview:hover {
		transform: scale(1.1);
	}

	.color-input {
		flex: 1;
		font-family: monospace;
		font-size: 1rem;
		padding: 0.5rem;
		border: 1px solid #dbdbdb;
		border-radius: 4px;
		text-transform: uppercase;
	}

	.color-input:focus {
		outline: none;
		border-color: #32b3d3;
	}

	.color-input.is-loading {
		border-color: #3298dc;
		opacity: 0.7;
	}

	.color-input.is-success {
		border-color: #48c774;
	}

	.color-input.is-danger {
		border-color: #f14668;
	}

	.hex-prefix {
		font-family: monospace;
		font-weight: bold;
		color: #666;
	}

	.reset-button {
		margin-top: 1rem;
	}

	.state-description {
		font-size: 0.875rem;
		color: #666;
		font-style: italic;
		margin-top: 0.25rem;
	}
</style>

{#if mounted}
<Box title={$_("config.titles.led")} icon="ic:outline-light-mode" back={true}>
	<div class="columns is-centered is-vcentered has-text-dark">
		<div class="column is-two-thirds">
			<div class="notification is-info is-light mb-4">
				<p>{$_("config.led.info")}</p>
			</div>

			{#each ledStates as state}
			<div class="my-3 is-flex is-justify-content-center">
				<Borders grow={true} has_help={true}>
					<div slot="help">
						{@html $_("config.led." + state.key + "-help")}
					</div>
					<div class="is-uppercase has-text-weight-bold is-size-6 mb-2">
						{$_(("config.led." + state.key))}
					</div>
					<div class="state-description">{state.desc}</div>
					<div class="color-input-wrapper mt-3">
						<input
							type="color"
							class="color-preview"
							value={"#" + formdata[state.key].val}
							on:input={(e) => {
								formdata[state.key].val = e.target.value.substring(1).toUpperCase()
								formdata = formdata // Trigger Svelte reactivity
							}}
							on:change={() => setProperty(state.key)}
							title={$_("config.led.color-picker")}
						/>
						<span class="hex-prefix">#</span>
						<input
							type="text"
							class="color-input"
							class:is-loading={formdata[state.key].status === "loading" || formdata[state.key].status === "testing"}
							class:is-success={formdata[state.key].status === "ok"}
							class:is-danger={formdata[state.key].status === "error"}
							maxlength="6"
							pattern="[0-9A-Fa-f]{6}"
							bind:value={formdata[state.key].val}
							bind:this={formdata[state.key].input}
							on:change={() => {
								// Validate hex input
								formdata[state.key].val = formdata[state.key].val.replace(/[^0-9A-Fa-f]/g, '').substring(0, 6).toUpperCase()
								if (formdata[state.key].val.length === 6) {
									setProperty(state.key)
								}
							}}
							placeholder="RRGGBB"
						/>
						<button 
							class="button is-small is-info ml-2" 
							on:click={() => testColor(state.key)}
							disabled={formdata[state.key].status === "testing" || formdata[state.key].status === "loading"}
							title="Test this color on LED strip for 3 seconds"
						>
							<span class="icon">
								<i class="fa fa-bolt"></i>
							</span>
							<span>Test</span>
						</button>
					</div>
				</Borders>
			</div>
			{/each}

			<div class="my-3 is-flex is-justify-content-center">
				<button class="button is-warning reset-button" on:click={resetToDefaults}>
					<span class="icon">
						<i class="fa fa-undo"></i>
					</span>
					<span>{$_("config.led.reset-defaults")}</span>
				</button>
			</div>
		</div>
	</div>
</Box>
{/if}
