<script lang="ts">
  import { onMount } from "svelte";
  import { Connection } from "@solana/web3.js";
  import { Provider, web3 } from "@project-serum/anchor";
  import { fade } from "svelte/transition";
  import Button from "./components/Button.svelte";
  import Header from "./components/CardHeader.svelte";
  import Discord from "./assets/discord.svg"

  import { candyMachineState, userState } from "./lib/store";
  import {
    getCandyMachineState,
    checkWalletConnected,
    getUserBalance,
    existsOwnerSPLToken,
  } from "./lib/state-helpers";

  /***********************************/
  // Customise the app by changing the following variables.
  const TITLE = "Spookies";
  const DESCRTIPTION = "A collection of spooky bois on the blockchain";
  const HEADER_TITLE = "spooky url ";
  const HEADER_LINK = "https://solarare.com";
  // Your image or GIF needs to be in the /public folder for this to work
  const IMAGE_LINK = "https://www.arsenalpics.com/p/5/thierry-henry-breaks-tottenham-defence-way-4379759.jpg.webp";
  /***********************************/

  let { solana } = window as any;
  const rpcUrl = "https://metaplex.devnet.rpcpool.com/"
  const cluster = "devnet"
  const candyMachineId = "31hvoGGGuw4Wf8a4Nn5FZDAXhGhK6cCQ55BnBopkJEoh"
  const opts = { preflightCommitment: "processed" };

  let siteLoading = true;
  let errorOcurred = false;
  let connection: Connection;
  let provider: Provider;
  let candyMachinePublicKey: web3.PublicKey;

  $: itemsRedeemed = $candyMachineState?.state.itemsRedeemed;
  $: itemsAvailable = $candyMachineState?.state.itemsAvailable;

  function checkEnvironmentVariables() {
    // Check if populated
    if (!rpcUrl || !candyMachineId || !cluster) {
      if (!rpcUrl) {
        console.error("RPC URL not populated");
      }
      if (!candyMachineId) {
        console.error("Candy Machine ID not populated");
      }
      if (!cluster) {
        console.error("Environment not populated");
      }
      return true;
    }
    if (candyMachineId.length < 32 || candyMachineId.length > 44) {
      console.error(
        "Candy Machine Public Key is invalid. Enter a length in-between 32 and 44 characters"
      );
      return true;
    }
    return false;
  }

  onMount(async () => {
    solana = (window as any).solana;
    // Check if environement variables are populated
    errorOcurred = checkEnvironmentVariables();
    if (errorOcurred) {
      return;
    }

    // If env variables populated, create provider, PK and connection
    connection = new Connection(rpcUrl);
    provider = new Provider(
      connection,
      solana,
      opts.preflightCommitment as web3.ConfirmOptions
    );
    candyMachinePublicKey = new web3.PublicKey(candyMachineId);

    // Get candy machine state
    $candyMachineState = await getCandyMachineState(
      candyMachinePublicKey,
      provider
    );
    // Establish connection to wallet
    if (solana?.isPhantom) {
      $userState.walletPublicKey = await checkWalletConnected(solana);
      if ($userState.walletPublicKey) {
        // Get User Balance
        $userState.userBalance = await getUserBalance(
          $userState.walletPublicKey,
          connection
        );
        // If whitelist config populated, check if user is whitelisted (ie. check if they have token)
        if ($candyMachineState.state.whitelistMintSettings) {
          $userState.isWhiteListed = await existsOwnerSPLToken(
            $userState.walletPublicKey,
            connection,
            $candyMachineState.state.whitelistMintSettings?.mint
          );
        }
      }
    }

    // Stop loading
    siteLoading = false;
  });
</script>

<main class="h-screen">
  <!-- Error section -->
  {#if errorOcurred}
    <div class=" h-full flex">
      <div class="m-auto">
        An error occurred. Please check if your environment variables have been
        populated correctly and redeploy the applcation.
      </div>
    </div>
    <!-- Loading Section -->
  {:else if siteLoading && !errorOcurred}
    <div class=" h-full flex">
      <div class="lds-hourglass m-auto" />
    </div>
  {:else}
    <!-- Menu Bar -->
    {#if HEADER_TITLE}
      <a
        href={HEADER_LINK}
        class="text-black tracking-widest underline underline-offset-4 decoration-2 font-mono"
        >{HEADER_TITLE}</a
      >
      <a
        href="https://github.com/matt-Yendys"
        >
      <img src={Discord} alt="" class="w-1/12 mx-auto m-2" />
      </a>
    {/if}
    <!-- Card -->
    <div
      class=" max-w-lg mx-auto bg-white rounded-lg border-2"
      transition:fade
    >
      <!-- Top Bar -->
      <Header />
      <hr />
      <br />
      <!-- Main Body -->
      <div class="p-6">
        <img src={IMAGE_LINK} alt="" class=" w-1/2 mx-auto m-5" />
        <div
          class=" text-lg sm:text-2xl font-mono font-bold py-5 tracking-wider"
        >
          {TITLE}
        </div>
        <div class="text-sm sm:text-md font-semibold pb-5 text-gray-600 ">
          {DESCRTIPTION}
        </div>
        <Button {connection} />

        <div class=" tracking-widest font-bold text-sm pt-3 text-gray-400">
          {itemsRedeemed}/{itemsAvailable} claimed
        </div>
        <div class="flex flex-col pt-3">
          {#if $userState.solanaExplorerLink}
            <a
              href={$userState.solanaExplorerLink}
              target="_blank"
              class="text-purple-700 font-semibold  p-1"
              >View on Solana Explorer</a
            >
          {/if}
        </div>
      </div>
    </div>
  {/if}
</main>
