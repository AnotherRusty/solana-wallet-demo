<script lang="ts">
    import "../app.css";
    import { onMount } from "svelte";

    import { getWallets } from "@wallet-standard/app";
    import {
        getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
        getOrCreateUiWalletAccountForStandardWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
    } from "@wallet-standard/ui-registry";
    import {
        getWalletFeature,
        type UiWalletAccount,
        type UiWallet,
    } from "@wallet-standard/ui";
    import {
        StandardConnect,
        StandardEvents,
        type StandardConnectFeature,
    } from "@wallet-standard/features";
    import type { Wallet } from "@wallet-standard/base";

    let account: UiWalletAccount | null = null;

    let installedUiWallets: UiWallet[] = [];

    const walletRegistry = getWallets();

    onMount(async () => {
        // get connected account
        await initSolanaWallets();
    });

    function copyAddress(address: string) {
        navigator.clipboard.writeText(address);
    }
    async function initSolanaWallets() {
        if (installedUiWallets.length > 0) return;
        // const installedWallets = getWallets();
        const installedWallets = walletRegistry.get();
        console.log("wallets", installedWallets);
        for (const wallet of installedWallets) {
            const uiWallet =
                getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(
                    wallet,
                );
            // const supportedUiWalletFeatures = getWalletFeature(uiWallet, StandardConnect);
            if (!uiWallet.chains.includes("solana:devnet")) {
                continue; // skip sui wallet or others
            }

            // switch account in browser wallet plugin will trigger this event
            wallet.features[StandardEvents].on("change", ({ accounts }) => {
                account = accounts[0];
            });

            console.log("solana uiWallet", uiWallet); // not autoConnect so first load will not have account
            await connectWallet(uiWallet);
            installedUiWallets = [...installedUiWallets, uiWallet];
            if (uiWallet.accounts.length > 0) {
                account = uiWallet.accounts[0];
                break;
            }
        }
    }

    async function connectWallet(wallet: UiWallet) {
        const connectFeature = getWalletFeature(
            wallet,
            StandardConnect,
        ) as StandardConnectFeature[typeof StandardConnect];

        const accountsPromise = connectFeature
            .connect()
            .then(({ accounts }) => {
                console.log("connected", accounts);
                return accounts;
            });
        await accountsPromise;
    }

    async function handleTransfer() {
        // ensure account exists
        if (!account) return;
        
    }
</script>

<div class="container mx-auto">
    {#if account}
        <div class="dropdown dropdown-end">
            <div tabindex="0" role="button" class="btn btn-primary m-1">
                {account.address}
            </div>
            <ul
                class="dropdown-content menu bg-base-100 rounded-box z-[1] w-52 p-2 shadow"
            >
                <li>
                    <button on:click={() => copyAddress(account?.address)}
                        >Copy Address</button
                    >
                </li>
            </ul>
        </div>
    {:else}
        <div class="dropdown dropdown-end">
            <div tabindex="0" role="button" class="btn btn-primary m-1">
                Select Wallet
            </div>
            <ul
                class="dropdown-content menu bg-base-100 rounded-box z-[1] w-52 p-2 shadow"
            >
                {#each installedUiWallets as wallet}
                    <li>
                        <button on:click={() => connectWallet(wallet)}
                            >{wallet.name}</button
                        >
                    </li>
                {/each}
            </ul>
        </div>
    {/if}

    {#if account}
    <div class="card bg-base-100 w-full shadow-xl">
        <div class="card-body">
            <h2 class="card-title">Transfer 0.1 SOL on devnet</h2>
            <p>receiver: FDjn87xPsLiXwakFygi4uEdet568o7A22UboxrUCwu7A</p>
            <div class="card-actions justify-end">
                <button on:click={() => handleTransfer()} class="btn btn-primary">Transfer</button>
            </div>
        </div>
    </div>
    {/if}
</div>
