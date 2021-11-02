<script lang="ts">
  import { onMount, afterUpdate } from "svelte";
  import { fly } from "svelte/transition";
  import { TezosToolkit, compose } from "@taquito/taquito";
  import { Tzip12Module, tzip12 } from "@taquito/tzip12";
  import { Tzip16Module, tzip16 } from "@taquito/tzip16";
  import { validateContractAddress } from "@taquito/utils";
  import { Parser, emitMicheline } from "@taquito/michel-codec";
  import { InMemorySigner } from "@taquito/signer";
  import Modal from "./Modal.svelte";

  export let params;

  type Network =
    | "mainnet"
    | "florencenet"
    | "granadanet"
    | "hangzhounet"
    | undefined;

  let Tezos: TezosToolkit;
  let parser: Parser;
  let contractAddress = "";
  let loadingMetadata = false;
  let contractAddressError = false;
  let errorMessage = "";
  let metadata = undefined;
  let views = undefined;
  let refreshViews = false;
  let modalPayload: {
    viewName: string;
    payload: string;
    param: any; // undefined or Michelson JSON
    execute: any;
  } = {
    viewName: "",
    payload: "",
    param: false,
    execute: null
  };
  let modalOpen = false;
  const tokenMetadataRegex = /\"token_metadata\":\"([0-9]+)\"/;
  let network: Network = "hangzhounet";
  const examples: { network: Network; address: string; text: string }[] = [
    {
      network: "hangzhounet",
      address: "KT1AekHCGmUU7ZdfBhEdrVJM34CzGRB3iEXF",
      text: "HTTPS"
    },
    {
      network: "hangzhounet",
      address: "KT1EPfHKY1oCjBaW4vGD9ResuXjdRsca3C5C",
      text: "HTTPS empty metadata"
    },
    {
      network: "hangzhounet",
      address: "KT1UWvoSuKuXPF6yaM1NDP3rKz9iV1RJx1RN",
      text: "HTTPS with emoji"
    },
    {
      network: "hangzhounet",
      address: "KT1C5m9xgjJeb96z636PCMrQNhVKj4ogd43S",
      text: "HTTPS invalida metadata"
    },
    {
      network: "hangzhounet",
      address: "KT1BwB2pkYy87NMmpsp2gBwzrNuo6dm4bXq9",
      text: "HTTPS with sha256"
    },
    // {
    //   network: "hangzhounet",
    //   address: "KT1E4sUuWvqC9pgCBEH9xxz6FhuNac11yR3o",
    //   text: "HTTPS invalid sha256"
    // },
    {
      network: "hangzhounet",
      address: "KT1Fh7B2gpBhYuQB5Ac1No42YDa6fvyqTyRC",
      text: "IPFS"
    },
    // {
    //   network: "hangzhounet",
    //   address: "KT1UhE8bubiWbKUb3RSZLDS2zz7gxkFuUecD",
    //   text: "Tezos Storage - metadata in current contract"
    // },
    // {
    //   network: "hangzhounet",
    //   address: "KT1QNprdGLFXvQwzAtDRT6MK1Pq5AdWQnHsf",
    //   text: "Tezos Storage - metadata in another contract"
    // },
    {
      network: "hangzhounet",
      address: "KT1QCnt2H2m9oTueZ4HTxAXHLZwi2iYw3otG",
      text: "Token Metadata in views"
    },
    {
      network: "hangzhounet",
      address: "KT1G4gTu78wYdNfGBDy75N1r9evq5iTM8Hy5",
      text: "Token Metadata in storage"
    }
  ];
  const rpcProviders = {
    mainnet: "https://mainnet.api.tez.ie",
    granadanet: "https://granadanet.api.tez.ie",
    florencenet: "https://florencenet.api.tez.ie",
    hangzhounet: "https://hangzhounet.api.tez.ie",
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
            .map(view => {
              let name: string;
              if (view.name) {
                name = view.name;
              } else {
                name = "N/A";
              }
              return `<details><summary><strong><em>${name}</em></strong>:${
                views && views.hasOwnProperty(name)
                  ? `<button class="execute-view-button" data-view="${view.name}">Execute</button>`
                  : ""
              }</summary><div>${parseObject(view)}</div></details>`;
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
        const contract = await Tezos.contract.at(
          contractAddress,
          compose(tzip16, tzip12)
        );
        views = await contract.tzip16().metadataViews();
        metadata = await contract.tzip16().getMetadata();
        console.log("metadata:", metadata);
        const storage: any = await contract.storage();
        if (views && views.hasOwnProperty("all_tokens")) {
          console.log("token metadata are in views:");
          const rawTokenIDs = await views.all_tokens().executeView();
          const tokenIDs = rawTokenIDs.map(tkID => tkID.toNumber());
          const promises = [];
          tokenIDs.forEach(tokenID => {
            promises.push(contract.tzip12().getTokenMetadata(tokenID));
          });
          const tokens = await Promise.all(promises);
          if (Array.isArray(tokens) && tokens.length > 0) {
            metadata.tokenMetadata = [...tokens];
          } else {
            metadata.tokenMetadata = undefined;
          }
        } else if (tokenMetadataRegex.test(JSON.stringify(storage))) {
          console.log("token metadata are in storage");
          const match = JSON.stringify(storage).match(tokenMetadataRegex);
          if (match) {
            // gets token ids from indexer
            const bigmapID = match[1].toString();
            const data = await fetch(
              `https://api.better-call.dev/v1/bigmap/${network}/${bigmapID}/keys`
            );
            if (data) {
              const json = await data.json();
              const tokenIDs: number[] = json.map(el => {
                if (!isNaN(el.data.key.value)) {
                  return +el.data.key.value;
                } else {
                  throw "Invalid token ID";
                }
              });
              const promises = [];
              tokenIDs.forEach(tokenID => {
                promises.push(contract.tzip12().getTokenMetadata(tokenID));
              });
              const tokens = await Promise.all(promises);
              if (Array.isArray(tokens) && tokens.length > 0) {
                metadata.tokenMetadata = [...tokens];
              } else {
                metadata.tokenMetadata = undefined;
              }
            } else {
              metadata.tokenMetadata = undefined;
            }
          }
        } else {
          metadata.tokenMetadata = undefined;
        }
        contractLink = `https://taquito-metadata-explorer.netlify.app/#/${network}/${contractAddress}`;
        refreshViews = true;
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
    Tezos = new TezosToolkit(rpcProviders[network]);
    Tezos.addExtension(new Tzip16Module());
    Tezos.addExtension(new Tzip12Module());
    parser = new Parser();

    const key =
      "edskRpm2mUhvoUjHjXgMoDRxMKhtKfww1ixmWiHCWhHuMEEbGzdnz8Ks4vgarKDtxok7HmrEo1JzkXkdkvyw7Rtw6BNtSd7MJ7";
    const signer = new InMemorySigner(key);
    Tezos.setSignerProvider(signer);
    // loads parameters from URL
    if (
      params.network &&
      params.contract &&
      ["mainnet", "florencenet", "edonet"].includes(
        params.network.toLowerCase()
      ) &&
      validateContractAddress(params.contract) === 3
    ) {
      network = params.network;
      Tezos.setRpcProvider(rpcProviders[params.network.toLowerCase()]);
      contractAddress = params.contract;
      loadMetadata();
    }
    refreshViews = true;
  });

  afterUpdate(() => {
    if (refreshViews && views && Object.keys(views).length > 0) {
      const buttons = document.getElementsByClassName("execute-view-button");
      if (buttons && buttons.length > 0) {
        Array.from(buttons).forEach(element => {
          element.addEventListener("click", async event => {
            const dataset = {
              ...(event.target as HTMLElement).dataset
            };
            if (
              dataset &&
              dataset.hasOwnProperty("view") &&
              views.hasOwnProperty(dataset.view)
            ) {
              // checks if the view takes a parameter
              const viewImplementations = metadata.metadata.views.filter(
                v => v.name === dataset.view
              )[0].implementations;
              if (
                viewImplementations.length > 0 &&
                viewImplementations[0].hasOwnProperty("michelsonStorageView") &&
                viewImplementations[0].michelsonStorageView.hasOwnProperty(
                  "parameter"
                )
              ) {
                const param =
                  viewImplementations[0].michelsonStorageView.parameter;
                if (param && param.hasOwnProperty("prim")) {
                  const payload = param.prim;
                  modalPayload = {
                    viewName: dataset.view,
                    payload,
                    param,
                    execute: views
                  };
                  modalOpen = true;
                }
              } else {
                // the view takes no parameter
                const payload = await views[dataset.view]().executeView();
                modalPayload = {
                  viewName: dataset.view,
                  payload,
                  param: undefined,
                  execute: null
                };
                modalOpen = true;
              }
            }
          });
        });
      }
      refreshViews = false;
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
        z-index: 100;
        max-height: 400px;
        overflow: auto;

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

  .error-message {
    color: red;
    width: 70%;
    word-break: break-word;
  }
</style>

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
      {#each Object.keys(rpcProviders) as networkName}
        <p
          on:click={() => {
            Tezos.setRpcProvider(rpcProviders[networkName]);
            network = networkName;
            expandAll = false;
          }}
        >
          {networkName[0].toUpperCase() + networkName.slice(1)}
        </p>
      {/each}
    </div>
  </div>
  <div class="form">
    <div class="examples-wrapper">
      <div class="examples">
        Examples &nbsp;
        <span id="examples-arrowhead-down">&#x25BC;</span>
      </div>
      <div class="examples-list">
        {#each examples.filter(ex => ex.network === network) as example, index}
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
        {:else}
          <p>No example</p>
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
    <p class="error-message">Invalid Contract Address</p>
  {/if}
  {#if errorMessage}
    <p class="error-message">{errorMessage}</p>
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
          {:else if property === "tokenMetadata" && Array.isArray(metadata.tokenMetadata) && metadata.tokenMetadata.length > 0}
            <details>
              <summary>
                <strong>TOKENS METADATA</strong>:
              </summary>
              <div>
                {#each metadata.tokenMetadata as tkmt}
                  <details>
                    <summary><strong>{tkmt["name"]}:</strong></summary>
                    {#each Object.keys(tkmt) as prop}
                      <div>{prop}: {tkmt[prop]}</div>
                    {/each}
                  </details>
                {/each}
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
                    style="font-size:0.6rem;font-weight:bold;margin-left:10px"
                  >
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
{#if modalOpen}
  <Modal
    payload={modalPayload}
    close={() => {
      modalOpen = false;
      modalPayload = {
        viewName: "",
        payload: "",
        param: undefined,
        execute: undefined
      };
    }}
  />
{/if}
