<script lang="ts">
  import { fly, fade } from "svelte/transition";

  export let close: (p?: any) => any | undefined; //should be a function
  export let payload;

  let viewParam = "";
  let resultRunView = "";
</script>

<style lang="scss">
  .modal-wrapper {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    width: 100%;
    display: grid;
    place-items: center;
  }

  .background {
    background-color: black;
    opacity: 0.5;
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    width: 100%;
  }

  .modal {
    background-color: white;
    position: relative;
    padding: 30px;
    border-radius: 10px;
    max-width: 40%;
    max-height: 600px;

    .modal__close {
      color: #374251;
      border: solid 4px #374251;
      border-radius: 50px;
      position: absolute;
      top: -15px;
      right: -15px;
      height: 28px;
      width: 28px;
      background-color: white;
      text-align: center;
      font-size: 20px;
      cursor: pointer;
      transition: 0.3s;
      z-index: 100;

      span {
        vertical-align: top;
      }

      &:hover {
        color: #f04444;
      }
    }

    .modal__body {
      margin-bottom: 30px;
      max-height: 400px;
      overflow: auto;
    }

    .modal__footer {
      width: 100%;
      display: flex;
      justify-content: flex-end;
    }
  }

  .button {
    outline: none;
    appearance: none;
    border: none;
    padding: 10px 20px;
    margin: 0px 10px;
    border-radius: 5px;
    transition: 0.3s;
    color: #4a5568;
    font-family: "Poppins", sans-serif;
    cursor: pointer;

    &.green {
      background-color: #c6f6d5;

      &:hover:enabled {
        background-color: darken(#c6f6d5, 5);
        cursor: pointer;
      }
    }

    /*&.red {
    background-color: #fed7d7;

    &:hover:enabled {
      background-color: darken(#fed7d7, 5);
      cursor: pointer;
    }
  }*/

    &.blue {
      background-color: #bee3f8;

      &:hover:enabled {
        background-color: darken(#bee3f8, 5);
        cursor: pointer;
      }
    }

    /*&.disabled {
    background-color: #edf2f7;

    &:hover:enabled {
      cursor: pointer;
    }
  }*/
  }
</style>

<div class="modal-wrapper">
  <div class="background" transition:fade={{ duration: 200 }} />
  <div class="modal" transition:fly={{ y: 100, duration: 500 }}>
    <div class="modal__close" on:click={close}><span>&#10006;</span></div>
    <div class="modal__body">
      {#if payload.param}
        {#if !resultRunView}
          <div>
            Enter a parameter to run the <br /><em>{payload.viewName}</em> view:
          </div>
          <br />
          <div>
            <input
              type="text"
              placeholder={`Parameter of type ${payload.payload}`}
              bind:value={viewParam}
            />
          </div>
        {:else}
          <div>Result for <em>{payload.viewName}</em> view:</div>
          <br />
          <div style="word-break:break-all">
            {JSON.stringify(resultRunView)}
          </div>
        {/if}
      {:else}
        <div>Running <em>{payload.viewName}</em> view.</div>
        <br />
        <div>Result:</div>
        <div style="word-break:break-all">
          {payload.payload ? payload.payload : "undefined"}
        </div>
      {/if}
    </div>
    <div class="modal__footer">
      {#if payload.param && !resultRunView}
        <button
          class="button green"
          on:click={async () => {
            const result = await payload.execute[
              payload.viewName
            ]().executeView(viewParam);
            if (result) {
              resultRunView = result;
            }
          }}>Execute</button
        >
      {/if}
      <button class="button blue" on:click={close}>Close</button>
    </div>
  </div>
</div>
