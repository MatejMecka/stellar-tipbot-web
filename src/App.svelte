<script lang="ts">
  import { onMount } from "svelte";
  import {
    StellarWalletsKit,
    WalletNetwork,
    allowAllModules,
    XBULL_ID
  } from '@creit.tech/stellar-wallets-kit';
  import * as StellarSdk from '@stellar/stellar-sdk';
  import { Turnstile } from 'svelte-turnstile';

  // Parameters required to execute transaction
  const urlParams = new URLSearchParams(window.location.search);
  let source = urlParams.get('source');
  let destination = urlParams.get('destination');
  let amount = urlParams.get('amount');
  let discord_user_id = urlParams.get('discord_id');
  const testnet = urlParams.get('testnet');
  const submit_client = urlParams.get('submit_client')

  let discord_username = "Beary Bear";
  let discord_image = "https://static.wikia.nocookie.net/webarebears/images/4/45/Grizzly_Bear_Standing.png/revision/latest?cb=20160620214616";

  let server;
  let networkPassphrase;
  let networkPassphraseWalletKit;
  let xdr;

  let successAlert = false;
  let errorAlert = false;
  let errorMessage = undefined;
  let isLoading = false;
  let message = "Submitting transaction to network...";

  let horizon_url = undefined;
  let stellar_expert_url = undefined;

  let turnstile_token = undefined;

  if(testnet?.toLowerCase() == "true") {
    console.log("USING TESTNET")
     server = new StellarSdk.Horizon.Server(
      "https://horizon-testnet.stellar.org",
    );
    networkPassphrase = StellarSdk.Networks.TESTNET
    networkPassphraseWalletKit = WalletNetwork.TESTNET
    stellar_expert_url = "https://stellar.expert/explorer/testnet/tx/"
  } else {
     server = new StellarSdk.Horizon.Server(
      "https://horizon.stellar.org",
    );
    networkPassphrase = StellarSdk.Networks.PUBLIC
    networkPassphraseWalletKit = WalletNetwork.PUBLIC
    stellar_expert_url = "https://stellar.expert/explorer/public/tx/"
  }

  // Wallet Kit
  const kit: StellarWalletsKit = new StellarWalletsKit({
    network: networkPassphraseWalletKit,
    selectedWalletId: XBULL_ID,
    modules: allowAllModules(),
  });

  // Fetch Discord Info
  onMount(async () => {
    await fetch(`https://discordlookup.mesalytic.moe/v1/user/${discord_user_id}`, {
      "method": "GET",
      "mode": "cors"
    })
    .then(response => response.json())
    .then(body => {
      console.log(body)
      discord_image = body["avatar"]["link"]
      discord_username = `@${body["username"]} - ${body["global_name"]}`
    }).catch((err) => {
      console.error(err)
      discord_user_id = undefined;
    })

    // Verify Source Destination Account Exists
    let sourceAccount;
    try {
    // Verify Source Account Exists
      sourceAccount = await server.loadAccount(source);
      console.log("Source Account Loaded:", sourceAccount); // Debugging output
    } catch (error) {
      source = undefined;
      if (error instanceof StellarSdk.NotFoundError) {
        throw new Error("The source account does not exist!");
      } else {
        throw error;
      }
    }

    // Verify Destination Address Exists
    server.loadAccount(destination)
      // If the account is not found, surface a nicer error message for logging.
    .catch(function (error) {
      destination = undefined
      if (error instanceof StellarSdk.NotFoundError) {
        throw new Error("The destination account does not exist!");
      } else return error;
    })

    // Build Transaction
    const transaction = new StellarSdk.TransactionBuilder(sourceAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: networkPassphrase,
    }).addOperation(
      StellarSdk.Operation.payment({
        destination: destination,
        asset: StellarSdk.Asset.native(),
        amount: `${amount}`,
      }),
    )
    .addMemo(StellarSdk.Memo.text("TipBot says Hello!"))
    .setTimeout(180)
    .build();
    
    xdr = transaction.toXDR();
    console.log(xdr)

})

  const handleTransaction = async function(){
    await kit.openModal({
      onWalletSelected: async (option: ISupportedWallet) => {
        kit.setWallet(option.id);
        const { signedTxXdr } = await kit.signTransaction(xdr, {
        source,
        networkPassphrase: networkPassphraseWalletKit
        });

      const signed_transaction = new StellarSdk.TransactionBuilder.fromXDR(
        signedTxXdr,
        networkPassphrase
      );

      isLoading = true;

      console.log(signed_transaction)

      try {
        if(submit_client?.toLowerCase() == "true") {
          const transactionResult = await server.submitTransaction(signed_transaction);
          console.log(JSON.stringify(transactionResult, null, 2));
          console.log('\nSuccess! View the transaction at: ');
          console.log(transactionResult._links.transaction.href);

          horizon_url = transactionResult._links.transaction.href
          stellar_expert_url += transactionResult.id
          successAlert = true
        } else {
          const body = {
            "discord_id": discord_user_id,
            "xdr": signed_transaction.toXDR(),
            "turnstile_token": turnstile_token
          }

          const options = {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body:JSON.stringify(body)
          };

          fetch(`${import.meta.env.VITE_WORKER_URL}/submitTX`, options)
            .then(response => response.json())
            .then(response => {
              if(response["success"] == true) {
                horizon_url = response["tx_url_horizon"]
                stellar_expert_url = response["stellar_expert_url"]
                successAlert = true
                isLoading = false;
              } else {
                errorMessage = response['error']['message']
                errorAlert = true;
                isLoading = false;
              }
              console.log(response)
            })
            .catch(err => {
              console.error(err)
              isLoading = false;
            });

        }
      } catch (e) {
        console.log('An error has occured:');
        console.log(e);
        errorMessage = e;
        errorAlert = true;
        isLoading = false;
      }

      }
    });

    
  }

  const setTurnstileToken = function(data) {
    console.log(`Retrieved turnstile token: ${data}`)
    console.log(data)
    turnstile_token = data["detail"]["token"];
  }

</script>

{#if successAlert}
<div class="bg-green-100 border-l-4 border-green-500 text-green-700 p-4 mb-10" role="alert">
  <p class="font-bold">Transaction succesful</p>
  <p>Your tip has been sent! Thank you for using Tip Bot 💜</p>
  <p>View transaction on <a href={horizon_url} class="text-cyan-700">Horizon</a></p>
  <p>View transaction on <a href={stellar_expert_url} class="text-cyan-700">Stellar Expert</a></p>
</div>
{/if}

{#if errorAlert}
<div class="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 mb-10" role="alert">
  <p class="font-bold">There was an error submitting!</p>
  <p>{errorMessage}</p>
</div>
{/if}

{#if isLoading}
<div class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-50">
  <div class="bg-white p-6 rounded-md shadow-lg sm:max-w-[425px]">
    <div class="flex flex-col items-center justify-center">
      <svg class="animate-spin h-20 w-20 mr-3 text-blue-500" xmlns="http://www.w3.org/2000/svg" width="5em" height="5em" viewBox="0 0 24 24"><path fill="currentColor" d="M12,4a8,8,0,0,1,7.89,6.7A1.53,1.53,0,0,0,21.38,12h0a1.5,1.5,0,0,0,1.48-1.75,11,11,0,0,0-21.72,0A1.5,1.5,0,0,0,2.62,12h0a1.53,1.53,0,0,0,1.49-1.3A8,8,0,0,1,12,4Z"><animateTransform attributeName="transform" dur="0.75s" repeatCount="indefinite" type="rotate" values="0 12 12;360 12 12"/></path></svg>
      <p class="text-lg font-semibold text-center">{message}</p>
    </div>
  </div>
</div>
{/if}

{#if discord_user_id == undefined || source == undefined || destination == undefined}
<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
  <div class="bg-white rounded-lg shadow-xl max-w-md w-full">
    <div class="p-6">
      <div class="flex items-center gap-2 text-red-600 mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
        {#if discord_user_id == undefined}
          <h2 class="text-xl font-semibold">User Not Found</h2>
        {:else if source == undefined}
          <h2 class="text-xl font-semibold">Source Address Not Found</h2>
        {:else if destination == undefined}
          <h2 class="text-xl font-semibold">Destination Address Not Found</h2>
        {/if}
      </div>
      <p class="text-gray-600 mb-4">
        {#if discord_user_id == undefined}
          We couldn't find the user you're looking for. This could be due to one of the following reasons:
        {:else if source == undefined || destination == undefined}
          We couldn't find the address. This could be due to one of the following reasons:
        {/if}
        </p>
      <ul class="list-disc pl-6 space-y-2 text-gray-600 mb-6">
        <li>You might have copied the URL wrong</li>
        {#if discord_user_id == undefined}
          <li>The user may have been deleted</li>
          <li>The user ID might be incorrect</li>
        {:else if source == undefined || destination == undefined}
          <li>The provided address was not found on the Stellar Network</li>
          <li>You are attempting to tip on a wrong network.</li>
        {/if}
        <li>There might be a temporary issue with our system</li>
      </ul>
      <div class="flex justify-end">
        <button
          class="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-offset-2"
        >
          Close
        </button>
      </div>
    </div>
  </div>
</div>
{:else}
<div class="min-h-screen w-full flex items-center justify-center bg-gradient-to-br from-primary/20 to-secondary/20">
  <div class="bg-white rounded-lg shadow-lg w-full max-w-md mx-4">
    <div class="p-8">
      <div class="flex flex-col items-center space-y-6">
        <div class="h-32 w-32 rounded-full overflow-hidden bg-gray-200">
          <img src={discord_image} alt="@johndoe" class="h-full w-full object-cover" />
        </div>
        <h2 class="text-3xl font-bold text-center">{discord_username}</h2>
        <div class="flex items-center justify-center w-full max-w-xs">
          <span class="text-2xl font-semibold mr-2">XLM</span>
          <input
            type="text"
            bind:value={amount}
            class="w-24 text-right text-2xl border rounded-md py-2 px-3 focus:outline-none focus:ring-2 focus:ring-primary"
            aria-label="Tip amount"
          />
        </div>
        {#if import.meta.env.VITE_ENABLE_WORKER.toLowerCase() == 'true'}
          <Turnstile 
          siteKey={import.meta.env.VITE_TURNSTILE_SITE_KEY}
          on:callback={(data) => setTurnstileToken(data)}
          />
        {/if}
        <button class="w-full max-w-xs text-lg py-6 bg-neutral-950 text-white rounded-md hover:bg-primary/90 transition-colors"   on:click={handleTransaction}>
          Tip
        </button>
      </div>
    </div>
  </div>
</div>
{/if}

<footer class="footer footer-center bg-base-300 text-base-content p-4">
  <aside>
    <p>Made with Love by Nesho 🐻</p>
  </aside>
</footer>

<style>

</style>
