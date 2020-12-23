<script lang="ts">
	import { onMount } from "svelte";
	import { fly } from "svelte/transition";
	import { TezosToolkit } from "@taquito/taquito";
	import { Tzip16Module, tzip16 } from "@taquito/tzip16";
	import { validateContractAddress } from "@taquito/utils";
	import { location } from "svelte-spa-router";

	export let params;

	type Network = "mainnet" | "carthagenet" | "delphinet" | undefined;

	let Tezos: TezosToolkit;
	let contractAddress = "";
	let loadingMetadata = false;
	let contractAddressError = false;
	let errorMessage = "";
	let metadata = undefined;
	let network: Network = "delphinet";
	const examples: { network: Network; address: string; text: string }[] = [
		{
			network: "carthagenet",
			address: "KT1FeMKGGvdWiA4r5RaucoEUAa8cTEXSSpCX",
			text: "Tezos Storage",
		},
		{
			network: "carthagenet",
			address: "KT1A1mR7zS8cWBehnf5wa6eY1SwCY6Teigne",
			text: "HTTPS",
		},
		{
			network: "carthagenet",
			address: "KT1PBndiMVyeptfQejZKYcSB6YmucaJdXVBQ",
			text: "IPFS",
		},
		{
			network: "delphinet",
			address: "KT1TLvewkn73Hb1YTDyX6pE6oD8qVKGTZax3",
			text: "Tezos Storage",
		},
		{
			network: "delphinet",
			address: "KT1GPDQvmV37orH1XH3SZmVVKFaMuzzqsmN7",
			text: "Tezos Storage",
		},
		{
			network: "delphinet",
			address: "KT1TLvewkn73Hb1YTDyX6pE6oD8qVKGTZax3",
			text: "Tezos Storage",
		},
		{
			network: "delphinet",
			address: "KT1Peb7x8DfBMnHyyzdSDgpSyAvaZXLuTz5g",
			text: "Invalid URI",
		},
		{
			network: "delphinet",
			address: "KT1Dkn2fHtjtfLJ6SeTRQ7BujKEPk1pGjBAE",
			text: "Tezos Storage",
		},
		{
			network: "delphinet",
			address: "KT191tWhzxUvx3ziu1sMYrDweZLrQfgbvGC5",
			text: "Tezos Storage",
		},
		{
			network: "delphinet",
			address: "KT1RyihALYEsVCcKP7Ya6teCHs9ii5ZHQxvj",
			text: "Tezos Storage",
		},
	];
	const rpcProviders = {
		mainnet: "https://mainnet-tezos.giganode.io",
		delphinet: "https://delphinet.smartpy.io", //"https://delphinet-tezos-giganode.io",
		carthagenet: "https://carthagenet.smartpy.io",
	};
	let expandAll = false;
	let contractLink = "";

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
				result += `<details><summary><strong><em>${el}</em></strong>:</summary><div>${parseObject(
					obj[el]
				)}</div></details>`;
			}
		}

		return result;
	};

	const loadMetadata = async () => {
		contractAddressError = false;
		errorMessage = "";
		metadata = undefined;
		loadingMetadata = true;
		expandAll = false;

		if (validateContractAddress(contractAddress) === 3) {
			try {
				const contract = await Tezos.contract.at(
					contractAddress,
					tzip16
				);
				const mt = await contract.tzip16().getMetadata();
				metadata = mt;
				console.log(mt);
				contractLink = `https://taquito-metadata-explorer.netlify.app/#/${network}/${contractAddress}`;
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

	const expandOrClose = () => {
		const details = document.getElementsByTagName("details");
		if (expandAll) {
			[...details].forEach((detail) => (detail.open = false));
			expandAll = false;
		} else {
			[...details].forEach((detail) => (detail.open = true));
			expandAll = true;
		}
	};

	const copyToClipboard = () => {
		const copyText = document.querySelector(
			"#contract-link"
		) as HTMLInputElement;
		copyText.select();
		document.execCommand("copy");
	};

	onMount(() => {
		Tezos = new TezosToolkit("https://testnet-tezos.giganode.io");
		Tezos.addExtension(new Tzip16Module());
		// loads parameters from URL
		if (
			params.network &&
			params.contract &&
			["mainnet", "carthagenet", "delphinet"].includes(
				params.network.toLowerCase()
			) &&
			validateContractAddress(params.contract) === 3
		) {
			network = params.network;
			Tezos.setRpcProvider(rpcProviders[params.network.toLowerCase()]);
			contractAddress = params.contract;
			loadMetadata();
		}
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
		align-items: center;
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

			& .examples span#examples-arrowhead-down {
				transition: 0.4s;
			}

			&:hover .examples span#examples-arrowhead-down {
				transform: rotate(180deg);
				transition: 0.4s;
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

		& .networks span#networks-arrowhead-down {
			transition: 0.4s;
		}

		&:hover .networks span#networks-arrowhead-down {
			transform: rotate(180deg);
			transition: 0.4s;
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

	.links {
		width: 60%;
		display: flex;
		justify-content: space-around;

		span {
			color: rgb(0, 100, 200);
			text-decoration: none;
			cursor: pointer;

			&:hover {
				text-decoration: underline;
			}
		}
	}

	#contract-link {
		width: 1px;
		position: absolute;
		left: -1000px;
		top: -1000px;
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
	<h3>Placeholder for Jev's text</h3>
	<div class="networks-wrapper">
		<div class="networks">
			{network.slice(0, 1).toUpperCase() + network.slice(1)}&nbsp;
			<span id="networks-arrowhead-down">&#x25BC;</span>
		</div>
		<div class="networks-list">
			<p
				on:click={() => {
					Tezos.setRpcProvider(rpcProviders.mainnet);
					network = 'mainnet';
					expandAll = false;
				}}>
				Mainnet
			</p>
			<p
				on:click={() => {
					Tezos.setRpcProvider(rpcProviders.delphinet);
					network = 'delphinet';
					expandAll = false;
				}}>
				Delphinet
			</p>
			<p
				on:click={() => {
					Tezos.setRpcProvider(rpcProviders.carthagenet);
					network = 'carthagenet';
					expandAll = false;
				}}>
				Carthagenet
			</p>
		</div>
	</div>
	<div class="form">
		<div class="examples-wrapper">
			<div class="examples">
				Examples &nbsp;
				<span id="examples-arrowhead-down">&#x25BC;</span>
			</div>
			<div class="examples-list">
				{#each examples as example}
					<p
						on:click={() => {
							contractAddress = example.address;
							network = example.network;
							Tezos.setRpcProvider(rpcProviders[example.network]);
							loadMetadata();
						}}>
						{example.text}
					</p>
				{/each}
			</div>
		</div>
		<input
			type="text"
			bind:value={contractAddress}
			placeholder="Enter the address of the contract here" />
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
		<div class="links">
			<div>
				<a
					href={`https://better-call.dev/${network.toLowerCase()}/${contractAddress}/storage`}
					target="_blank"
					rel="noopener noreferrer nofollower">See contract storage</a>
			</div>
			<div>
				<span
					on:click={expandOrClose}>{expandAll ? 'Collapse all' : 'Expand all'}</span>
			</div>
			<div>
				<span on:click={copyToClipboard}>Share a link to this contract</span>
				<input type="text" id="contract-link" value={contractLink} />
			</div>
		</div>
		<div class="metadata-display">
			{#each Object.keys(metadata) as property}
				<div class="metadata">
					{#if property === 'metadata'}
						<details>
							<summary>
								<strong>{property.toUpperCase()}</strong>:
							</summary>
							<div>
								{@html parseObject(metadata.metadata)}
							</div>
						</details>
						<!--<div><strong>{property.toUpperCase()}</strong>:</div>
						<div>
							{@html parseObject(metadata.metadata)}
						</div>-->
					{:else if property === 'uri'}
						<details>
							<summary>
								<strong>{property.toUpperCase()}</strong>:
							</summary>
							<div class="metadata__details">
								{@html matchURL(metadata[property])}
							</div>
						</details>
						<!--<div><strong>{property.toUpperCase()}</strong>:</div>
						<div class="metadata__details">
							{@html matchURL(metadata[property])}
						</div>-->
					{:else}
						<details>
							<summary>
								<strong>{property.toUpperCase()}</strong>:
							</summary>
							<div class="metadata__details">
								{metadata[property]}
							</div>
						</details>
						<!--<div><strong>{property.toUpperCase()}</strong>:</div>
						<div class="metadata__details">
							{metadata[property]}
						</div>-->
					{/if}
				</div>
			{/each}
		</div>
	{/if}
</main>