<script lang="ts">
  import { onMount } from "svelte";
  import { fly } from "svelte/transition";
  import { TezosToolkit } from "@taquito/taquito";
  import { Tzip16Module, tzip16 } from "@taquito/tzip16";
  import { validateContractAddress } from "@taquito/utils";
  import { Parser, emitMicheline } from "@taquito/michel-codec";
  import { InMemorySigner } from "@taquito/signer";

  export let params;

  type Network = "mainnet" | "carthagenet" | "delphinet" | undefined;

  let Tezos: TezosToolkit;
  let parser: Parser;
  let contractAddress = "";
  let loadingMetadata = false;
  let contractAddressError = false;
  let errorMessage = "";
  let metadata = undefined;
  let views = undefined;
  let network: Network = "delphinet";
  const examples: { network: Network; address: string; text: string }[] = [
    {
      network: "delphinet",
      address: "KT1KGkToC8UUJBJLqHcLRkv7xvjWd8JwUuTo",
      text: "HTTPS"
    },
    {
      network: "delphinet",
      address: "KT1WTGDQ9j2mFE7SbgmoixNAVXH1ynjdagon",
      text: "HTTPS empty metadata"
    },
    {
      network: "delphinet",
      address: "KT194AJC8UQPguynGdJfEVynF9wfUghDjHSt",
      text: "HTTPS with emoji"
    },
    {
      network: "delphinet",
      address: "KT1UQyKUoCat9oQNHPGMDypQ4mWW44DFWzXt",
      text: "HTTPS invalida metadata"
    },
    {
      network: "delphinet",
      address: "KT1PHNmaHvQNjt1LTqdWobJUi2aeDeWUdQUq",
      text: "HTTPS with sha256"
    },
    {
      network: "delphinet",
      address: "KT1Bhj5fgQioJYnFbg8jeki5SgRd7ZsCfhwp",
      text: "HTTPS invalid sha256"
    },
    {
      network: "delphinet",
      address: "KT1BfdzrP3ybxSbQCNZrmdk2Y5AQjRK1KKkz",
      text: "IPFS"
    },
    {
      network: "delphinet",
      address: "KT1TLvewkn73Hb1YTDyX6pE6oD8qVKGTZax3",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT1GPDQvmV37orH1XH3SZmVVKFaMuzzqsmN7",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT1TLvewkn73Hb1YTDyX6pE6oD8qVKGTZax3",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT1Peb7x8DfBMnHyyzdSDgpSyAvaZXLuTz5g",
      text: "Invalid URI"
    },
    {
      network: "delphinet",
      address: "KT1Dkn2fHtjtfLJ6SeTRQ7BujKEPk1pGjBAE",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT191tWhzxUvx3ziu1sMYrDweZLrQfgbvGC5",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT1RyihALYEsVCcKP7Ya6teCHs9ii5ZHQxvj",
      text: "Tezos Storage"
    },
    {
      network: "delphinet",
      address: "KT1KTkzGMHN4P1XvT4X1kFT5ubcvzxs6ZfSq",
      text: "Tezos Storage - metadata in current contract"
    },
    {
      network: "delphinet",
      address: "KT1BAQ3nEsLrEeZdkij8KiekaWUVQERNF1Hi",
      text: "Tezos Storage - metadata in another contract"
    }
  ];
  const rpcProviders = {
    mainnet: "https://mainnet-tezos.giganode.io",
    delphinet: "https://testnet-tezos.giganode.io", // "https://delphinet.smartpy.io",
    carthagenet: "https://carthagenet.smartpy.io"
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
        `<a href="${url[0]}" target="_blank" rel="noreferrer noopener nofollow">${url[0]}</a>`
      );
    } else {
      // looks for ipfs hashes
      const ipfsRegex = new RegExp(/ipfs:\/\/([a-zA-Z-0-9]{46})/, "g");
      const ipfs = str.match(ipfsRegex);
      if (ipfs) {
        return str.replace(
          ipfs[0],
          `<a href="https://ipfs.io/ipfs/${ipfs[0].replace(
            "ipfs://",
            ""
          )}" target="_blank" rel="noreferrer noopener nofollow">${ipfs[0]}</a>`
        );
      } else {
        return str;
      }
    }
  };

  const parseObject = obj => {
    let result = "";

    for (let el in obj) {
      if (el === "bytes") {
        //console.log(obj[el]);
      }
      // if string
      if (typeof obj[el] === "string") {
        result += `<div class="metadata__details"><div><strong><em>${el}</em></strong>:</div><div>${matchURL(
          obj[el]
        )}</div></div>`;
      } else if (Array.isArray(obj[el]) && el === "authors") {
        result += `<div class="metadata__details"><div><strong><em>${el}</em></strong>:</div><div>${obj[
          el
        ]
          .map(author => matchURL(author))
          .join(" / ")}</div></div>`;
      } else if (typeof obj[el] === "object") {
        if (el === "views" && Array.isArray(obj[el])) {
          result += `<details><summary><strong><em>${el}</em></strong>:</summary><div>${obj[
            el
          ]
            .map((view, i) => {
              let name: string | number;
              if (view.name) {
                name = view.name;
                delete view.name;
              } else {
                name = i;
              }
              return `<details><summary><strong><em>${name}</em></strong>:</summary><div>${parseObject(
                view
              )}</div></details>`;
            })
            .join("")}</div></details>`;
        } else {
          // prettyprints JSON Michelson
          try {
            const code = parser.parseJSON(obj[el]);
            result += `<details><summary><strong><em>${el}</em></strong>:</summary><div>${emitMicheline(
              code
            )}</div></details>`;
          } catch (error) {
            //console.log(el, obj[el]);
            result += `<details><summary><strong><em>${el}</em></strong>:</summary><div>${parseObject(
              obj[el]
            )}</div></details>`;
          }
        }
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
        const contract = await Tezos.contract.at(contractAddress, tzip16);
        metadata = await contract.tzip16().getMetadata();
        console.log(metadata);
        views = await contract.tzip16().metadataViews();
        console.log(views);
        console.log(await views.someJson().executeView());
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
      [...details].forEach(detail => (detail.open = false));
      expandAll = false;
    } else {
      [...details].forEach(detail => (detail.open = true));
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
    parser = new Parser();

    const key =
      "edskRpm2mUhvoUjHjXgMoDRxMKhtKfww1ixmWiHCWhHuMEEbGzdnz8Ks4vgarKDtxok7HmrEo1JzkXkdkvyw7Rtw6BNtSd7MJ7";
    const signer = new InMemorySigner(key);
    Tezos.setSignerProvider(signer);
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

<main>
  <div class="title">
    <img
      src="images/taquito.png"
      alt="Taquito"
      in:fly={{ duration: 1500, delay: 600, x: 1000 }}
    />
    <h1>Taquito Metadata Explorer</h1>
    <img
      src="images/taquito.png"
      alt="Taquito"
      in:fly={{ duration: 1500, delay: 600, x: -1000 }}
    />
  </div>
  <h3>Explore TZIP-16 based Metadata associated with on chain contracts</h3>
  <div class="networks-wrapper">
    <div class="networks">
      {network.slice(0, 1).toUpperCase() + network.slice(1)}&nbsp;
      <span id="networks-arrowhead-down">&#x25BC;</span>
    </div>
    <div class="networks-list">
      <p
        on:click={() => {
          Tezos.setRpcProvider(rpcProviders.mainnet);
          network = "mainnet";
          expandAll = false;
        }}
      >Mainnet</p>
      <p
        on:click={() => {
          Tezos.setRpcProvider(rpcProviders.delphinet);
          network = "delphinet";
          expandAll = false;
        }}
      >Delphinet</p>
      <p
        on:click={() => {
          Tezos.setRpcProvider(rpcProviders.carthagenet);
          network = "carthagenet";
          expandAll = false;
        }}
      >Carthagenet</p>
    </div>
  </div>
  <div class="form">
    <div class="examples-wrapper">
      <div class="examples">
        Examples &nbsp;
        <span id="examples-arrowhead-down">&#x25BC;</span>
      </div>
      <div class="examples-list">
        {#each examples as example, index}
          <p
            on:click={() => {
              contractAddress = example.address;
              network = example.network;
              Tezos.setRpcProvider(rpcProviders[example.network]);
              loadMetadata();
            }}
            style={index === examples.length - 1
              ? "border-bottom-left-radius: 20px;border-bottom-right-radius: 20px;"
              : ""}
          >
            {example.text}
          </p>
        {/each}
      </div>
    </div>
    <input
      type="text"
      bind:value={contractAddress}
      placeholder="Enter the address of the contract here"
    />
    <button on:click={loadMetadata}>
      {#if loadingMetadata}
        <img src="images/taquito.png" alt="Taquito" class="taquito-button" />
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
          rel="noopener noreferrer nofollower">See contract storage</a
        >
      </div>
      <div>
        <span on:click={expandOrClose}
          >{expandAll ? "Collapse all" : "Expand all"}</span
        >
      </div>
      <div>
        <span on:click={copyToClipboard}>Share a link to this contract</span>
        <input type="text" id="contract-link" value={contractLink} />
      </div>
    </div>
    <div class="metadata-display">
      {#each Object.keys(metadata) as property}
        <div class="metadata">
          {#if property === "metadata"}
            <details>
              <summary>
                <strong>{property.toUpperCase()}</strong>:
              </summary>
              <div>
                {@html parseObject(metadata.metadata)}
              </div>
            </details>
          {:else if property === "uri"}
            <details>
              <summary>
                <strong>{property.toUpperCase()}</strong>:
              </summary>
              <div class="metadata__details">
                {@html matchURL(metadata[property])}
              </div>
            </details>
          {:else if property === "integrityCheckResult"}
            <details>
              <summary>
                <strong>{property.toUpperCase()}</strong>
                {#if metadata[property]}
                  <span class="integrety-test-result success">
                    &#10004;&#65039;
                  </span>:
                {:else if metadata[property] === undefined}
                  <span
                    class="integrety-test-result success"
                    style="font-size:0.6rem;font-weight:bold;margin-left:10px">
                    N/A
                  </span>:
                {:else}
                  <span class="integrety-test-result success"> &#10007; </span>:
                {/if}
              </summary>
              <div class="metadata__details">
                {metadata[property]}
              </div>
            </details>
          {:else}
            <details>
              <summary>
                <strong>{property.toUpperCase()}</strong>:
              </summary>
              <div class="metadata__details">
                {metadata[property]}
              </div>
            </details>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</main>

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
        z-index: 100;

        &:hover {
          display: block;
        }

        p {
          text-align: center;
          margin: 0px;
          padding: 5px 10px;
          cursor: pointer;

          &:hover {
            background-color: lighten(rgba(229, 231, 235, 1), 5);
          }
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
    padding: 15px 0px;

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

  .integrety-test-result {
    display: flex;
    justify-content: center;
    align-items: center;
    margin: 0px 5px;
    font-size: 1rem;
  }
</style>
