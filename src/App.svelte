<script lang="ts">
	import { onMount } from "svelte";
	import { fly } from "svelte/transition";
	import { TezosToolkit } from "@taquito/taquito";
	import { Tzip16Module, tzip16 } from "@taquito/tzip16";
	import { validateContractAddress } from "@taquito/utils";

	type Network = "Mainnet" | "Carthagenet" | "Delphinet" | undefined;

	let Tezos: TezosToolkit;
	let contractAddress = "";
	let loadingMetadata = false;
	let contractAddressError = false;
	let errorMessage = "";
	let metadata = undefined;
	const examples = [
		"KT1TLvewkn73Hb1YTDyX6pE6oD8qVKGTZax3",
		"KT1GPDQvmV37orH1XH3SZmVVKFaMuzzqsmN7",
	];
	let network: Network = "Delphinet";

	const matchURL = (str: string): string => {
		const regex = new RegExp(
			/https?:\/\/(www.)?[-a-zA-Z0-9@:%._+~#=]{1,256}.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_+.~#?&\/\/=]*)/,
			"g"
		);

		const url = str.match(regex);
		if (url) {
			return str.replace(
				url[0],
				`<a href="${url[0]}" target="_blank" rel="noreferrer noopener nofollower">${url[0]}</a>`
			);
		} else {
			return str;
		}
	};

	const parseObject = (obj) => {
		let result = "";

		for (let el in obj) {
			// if string
			if (typeof obj[el] === "string") {
				result += `<div class="metadata__details"><div><strong><em>${el}</em></strong>:</div><div>${matchURL(
					obj[el]
				)}</div></div>`;
			} else if (Array.isArray(obj[el]) && el === "authors") {
				result += `<div class="metadata__details"><div><strong><em>${el}</em></strong>:</div><div>${obj[
					el
				]
					.map((author) => matchURL(author))
					.join(" / ")}</div></div>`;
			} else if (typeof obj[el] === "object") {
				result += `<div class="metadata__details"><div><strong><em>${el}</em></strong>:</div><div>${parseObject(
					obj[el]
				)}</div></div>`;
			}
		}

		return result;
	};

	const loadMetadata = async () => {
		contractAddressError = false;
		errorMessage = "";
		metadata = undefined;
		loadingMetadata = true;

		if (validateContractAddress(contractAddress) === 3) {
			try {
				const contract = await Tezos.contract.at(
					contractAddress,
					tzip16
				);
				const mt = await contract.tzip16().getMetadata();
				metadata = mt;
				console.log(mt);
			} catch (error) {
				console.log(error);
				if (error.message) {
					errorMessage = error.message;
				} else {
					errorMessage = JSON.stringify(error, null, 2);
				}
			} finally {
				loadingMetadata = false;
			}
		} else {
			contractAddressError = true;
			loadingMetadata = false;
		}
	};

	onMount(() => {
		Tezos = new TezosToolkit("https://testnet-tezos.giganode.io");
		//Tezos = new TezosToolkit("https://carthagenet.smartpy.io");
		Tezos.addExtension(new Tzip16Module());
	});
</script>

<style lang="scss">
	$border-radius: 20px;

	main {
		display: flex;
		flex-direction: column;
		align-items: center;
		height: 100vh;
	}

	.title {
		display: flex;
		justify-content: space-between;
		text-align: center;

		img {
			$scale: 0.3;

			transform: scale($scale, $scale);
			-ms-transform: scale($scale, $scale);
			-webkit-transform: scale($scale, $scale);
		}
	}

	.form {
		display: flex;
		justify-content: center;
		margin-top: 20px;

		input,
		button,
		.examples {
			padding: 0px 20px;
			height: 65px;
			font-size: 1.5rem;
			border: solid 1px rgba(156, 163, 175, 1);
			outline: none;
			font-family: "Poppins", sans-serif;
		}

		input {
			width: 550px;
			border-top-left-radius: $border-radius;
			border-bottom-left-radius: $border-radius;
			border-right: none;
		}

		button {
			cursor: pointer;
			border-top-right-radius: $border-radius;
			border-bottom-right-radius: $border-radius;
			border-left: none;
			background-color: rgba(229, 231, 235, 1);
			height: 67px;
		}

		.examples-wrapper {
			position: relative;
			padding: 0px;
			margin: 0px;
			margin-right: 10px;

			&:hover .examples span#examples-arrowhead-up {
				display: inline;
			}
			&:hover .examples span#examples-arrowhead-down {
				display: none;
			}

			&:hover .examples {
				border-bottom-left-radius: 0px;
				border-bottom-right-radius: 0px;
			}

			.examples {
				border-radius: $border-radius;
				background-color: rgba(229, 231, 235, 1);
				cursor: pointer;
				display: flex;
				justify-content: center;
				align-items: center;
				transition: 0.3s;

				span#examples-arrowhead-up {
					display: none;
				}

				&:hover + .examples-list {
					display: block;
				}
			}

			.examples-list {
				display: none;
				margin: 0px;
				margin-top: 65px;
				position: absolute;
				top: 0px;
				left: 0px;
				background-color: rgba(229, 231, 235, 1);
				border: solid 1px rgba(156, 163, 175, 1);
				border-bottom-left-radius: $border-radius;
				border-bottom-right-radius: $border-radius;
				width: 99%;
				transition: 0.3s;

				&:hover {
					display: block;
				}

				p {
					text-align: center;
					margin: 0px;
					padding: 5px 10px;
					cursor: pointer;
				}
			}
		}
	}

	.taquito-button {
		width: 70px;
		-webkit-animation: pulsate-fwd 0.5s ease-in-out infinite both;
		animation: pulsate-fwd 0.5s ease-in-out infinite both;
	}

	.networks-wrapper {
		position: relative;

		&:hover .networks span#networks-arrowhead-up {
			display: inline;
		}
		&:hover .networks span#networks-arrowhead-down {
			display: none;
		}

		&:hover .networks {
			border-bottom-left-radius: 0px;
			border-bottom-right-radius: 0px;
		}

		.networks {
			border-radius: $border-radius;
			background-color: rgba(229, 231, 235, 1);
			border: solid 1px rgba(156, 163, 175, 1);
			padding: 0px 20px;
			cursor: pointer;
			display: flex;
			justify-content: center;
			align-items: center;
			transition: 0.3s;
			height: 30px;

			span#networks-arrowhead-up {
				display: none;
			}

			&:hover + .networks-list {
				display: block;
			}
		}

		.networks-list {
			display: none;
			margin: 0px;
			margin-top: 30px;
			position: absolute;
			top: 0px;
			left: 0px;
			background-color: rgba(229, 231, 235, 1);
			border: solid 1px rgba(156, 163, 175, 1);
			border-bottom-left-radius: $border-radius;
			border-bottom-right-radius: $border-radius;
			width: 98.5%;
			transition: 0.3s;

			&:hover {
				display: block;
			}

			p {
				text-align: center;
				margin: 0px;
				padding: 5px 10px;
				cursor: pointer;
			}
		}
	}
</style>

<main>
	<div class="title">
		<img
			src="images/taquito.png"
			alt="Taquito"
			in:fly={{ duration: 1500, delay: 600, x: 1000 }} />
		<h1>Taquito Metadata Explorer</h1>
		<img
			src="images/taquito.png"
			alt="Taquito"
			in:fly={{ duration: 1500, delay: 600, x: -1000 }} />
	</div>
	<div class="networks-wrapper">
		<div class="networks">
			{network}&nbsp;
			<span id="networks-arrowhead-down">&#x25BC;</span><span
				id="networks-arrowhead-up">&#x25B2;</span>
		</div>
		<div class="networks-list">
			<p
				on:click={() => {
					Tezos.setRpcProvider('https://mainnet-tezos.giganode.io');
					network = 'Mainnet';
				}}>
				Mainnet
			</p>
			<p
				on:click={() => {
					Tezos.setRpcProvider('https://delphinet-tezos-giganode.io');
					network = 'Delphinet';
				}}>
				Delphinet
			</p>
			<p
				on:click={() => {
					Tezos.setRpcProvider('https://carthagenet.smartpy.io');
					network = 'Carthagenet';
				}}>
				Carthagenet
			</p>
		</div>
	</div>
	<h3>Enter below the address of the contract:</h3>
	<div class="form">
		<div class="examples-wrapper">
			<div class="examples">
				Examples &nbsp;
				<span id="examples-arrowhead-down">&#x25BC;</span><span
					id="examples-arrowhead-up">&#x25B2;</span>
			</div>
			<div class="examples-list">
				{#each examples as example}
					<p
						on:click={() => {
							contractAddress = example;
							loadMetadata();
						}}>
						{example.slice(0, 7) + '...' + example.slice(-7)}
					</p>
				{/each}
			</div>
		</div>
		<input type="text" bind:value={contractAddress} />
		<button on:click={loadMetadata}>
			{#if loadingMetadata}
				<img
					src="images/taquito.png"
					alt="Taquito"
					class="taquito-button" />
			{:else}&nbsp;Load&nbsp;{/if}
		</button>
	</div>
	{#if contractAddressError}
		<p style="color:red">Invalid Contract Address</p>
	{/if}
	{#if errorMessage}
		<p style="color:red">{errorMessage}</p>
	{/if}
	<br />
	{#if metadata}
		<div class="metadata-display">
			{#each Object.keys(metadata) as property}
				<div class="metadata">
					{#if property === 'metadata'}
						<div><strong>{property.toUpperCase()}</strong>:</div>
						<div>
							{@html parseObject(metadata.metadata)}
						</div>
					{:else if property === 'uri'}
						<div><strong>{property.toUpperCase()}</strong>:</div>
						<div class="metadata__details">
							{@html matchURL(metadata[property])}
						</div>
					{:else}
						<div><strong>{property.toUpperCase()}</strong>:</div>
						<div class="metadata__details">
							{metadata[property]}
						</div>
					{/if}
				</div>
			{/each}
		</div>
	{/if}
</main>
