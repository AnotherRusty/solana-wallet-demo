<script lang="ts">
    import "../app.css";

    import { getWallets } from "@wallet-standard/app";
    import {
        getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
        getWalletForHandle_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
        getOrCreateUiWalletAccountForStandardWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
        getWalletAccountForUiWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
    } from "@wallet-standard/ui-registry";
    import {
        getWalletFeature,
        getWalletAccountFeature,
        type UiWalletAccount,
        type UiWallet,
    } from "@wallet-standard/ui";
    import {
        getTransactionEncoder,
        type Transaction,
    } from "@solana/transactions";
    import { type TransactionSendingSignerConfig } from "@solana/signers";
    import { getAbortablePromise } from "@solana/promises";
    import { type SignatureBytes } from "@solana/keys";

    import {
        StandardConnect,
        StandardDisconnect,
        StandardEvents,
        type StandardConnectFeature,
        type StandardDisconnectFeature,
        type StandardEventsFeature,
    } from "@wallet-standard/features";
    import type {
        IdentifierString,
        Wallet,
        WalletWithFeatures,
    } from "@wallet-standard/base";
    import {
        address,
        appendTransactionMessageInstruction,
        createTransactionMessage,
        lamports,
        pipe,
        setTransactionMessageFeePayerSigner,
        setTransactionMessageLifetimeUsingBlockhash,
        signAndSendTransactionMessageWithSigners,
        createSolanaRpc,
        createSolanaRpcSubscriptions,
        devnet,
    } from "@solana/web3.js";

    import { getTransferSolInstruction } from "@solana-program/system";

    // defined in: https://github.com/anza-xyz/wallet-standard/blob/master/packages/core/features/src/signAndSendTransaction.ts#L10C1-L11C1
    import {
        SolanaSignAndSendTransaction,
        type SolanaSignAndSendTransactionFeature,
    } from "@solana/wallet-standard-features";

    const { get } = getWallets();

    $: outputWallets = get();

    $: {
        outputWallets
            .filter(walletHasSignAndSendFeature)
            .map(subscribeToWalletEvents);
    }

    $: installedUiWallets = outputWallets.map((wallet) =>
        getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(
            wallet,
        ),
    );

    function walletHasSignAndSendFeature(
        wallet: Wallet,
    ): wallet is WalletWithFeatures<StandardEventsFeature> {
        return SolanaSignAndSendTransaction in wallet.features;
    }
    function subscribeToWalletEvents(
        wallet: WalletWithFeatures<StandardEventsFeature>,
    ): () => void {
        const dispose = wallet.features[StandardEvents].on(
            "change",
            ({ accounts }) => {
                console.log("wallet change listened", outputWallets, accounts);
                const newAccounts = accounts?.map(
                    getOrCreateUiWalletAccountForStandardWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED.bind(
                        null,
                        wallet,
                    ),
                );
                console.log("new Accounts", newAccounts);
                if (newAccounts && newAccounts.length > 0)
                    connectedWalletAccount = newAccounts[0];
                return newAccounts;
            },
        );
        return dispose;
    }

    function copyAddress(address?: string) {
        if (address) navigator.clipboard.writeText(address);
    }

    let connectedWallet: UiWallet | null = null;

    let connectedWalletAccount: UiWalletAccount | null = null;

    function uiWalletAccountsAreSame(
        a: UiWalletAccount,
        b: UiWalletAccount,
    ): boolean {
        if (a.address !== b.address) {
            return false;
        }
        const underlyingWalletA =
            getWalletForHandle_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(a);
        const underlyingWalletB =
            getWalletForHandle_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(b);
        return underlyingWalletA === underlyingWalletB;
    }
    async function connectWallet(uiWallet: UiWallet) {
        console.log("connect ui wallet", uiWallet);
        const wallet =
            getWalletForHandle_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(uiWallet);

        const existingAccounts = [...uiWallet.accounts];

        const connectFeature = getWalletFeature(
            uiWallet,
            StandardConnect,
        ) as StandardConnectFeature[typeof StandardConnect];

        const accountsPromise = connectFeature
            .connect()
            .then(({ accounts }) => {
                console.log("connected", accounts);
                // return accounts;
                return accounts.map(
                    getOrCreateUiWalletAccountForStandardWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED.bind(
                        null,
                        wallet,
                    ),
                ) as readonly UiWalletAccount[];
            });
        const nextAccounts = await accountsPromise;
        console.log("next accounts", nextAccounts);
        connectedWallet = uiWallet;

        // Try to choose the first never-before-seen account.
        for (const nextAccount of nextAccounts) {
            if (
                !existingAccounts.some((existingAccount) =>
                    uiWalletAccountsAreSame(nextAccount, existingAccount),
                )
            ) {
                // onAccountSelect(nextAccount);
                connectedWalletAccount = nextAccount;
                console.log("compare accounts", connectedWalletAccount);
                return;
            }
        }
        if (nextAccounts.length > 0) {
            // onAccountSelect(nextAccounts[0]);
            connectedWalletAccount = nextAccounts[0];
            console.log("default accounts", connectedWalletAccount);
        }
        // connectedWallet.accounts = newAccounts;
        // console.log('connected wallet', connectedWallet.accounts)
    }

    async function disconnectWallet() {
        if (!connectedWallet) return;
        const disconnectFeature = getWalletFeature(
            connectedWallet,
            StandardDisconnect,
        ) as StandardDisconnectFeature[typeof StandardDisconnect];
        await disconnectFeature.disconnect();
    }

    async function handleTransfer() {
        // ensure account exists
        console.log("handle transfer", connectedWalletAccount);
        if (!connectedWalletAccount) return;

        const receiver = address(
            "FDjn87xPsLiXwakFygi4uEdet568o7A22UboxrUCwu7A",
        );

        const rpc = createSolanaRpc(devnet("https://api.devnet.solana.com"));
        const rpcSubscriptions = createSolanaRpcSubscriptions(
            devnet("wss://api.devnet.solana.com"),
        );

        const { value: latestBlockhash } = await rpc
            .getLatestBlockhash({ commitment: "confirmed" })
            .send();

        console.log(
            "uiWallets & account",
            installedUiWallets,
            connectedWalletAccount,
        );
        const signAndSendTransactionFeature = getWalletAccountFeature(
            connectedWalletAccount,
            SolanaSignAndSendTransaction,
        ) as SolanaSignAndSendTransactionFeature[typeof SolanaSignAndSendTransaction];
        console.log(
            "signAndSendTransactionFeature",
            signAndSendTransactionFeature,
        );

        const transactionSendingSigner = {
            address: address(connectedWalletAccount.address),
            async signAndSendTransactions(
                transactions: Transaction[],
                config: TransactionSendingSignerConfig = {},
            ) {
                const { abortSignal, ...options } = config;
                abortSignal?.throwIfAborted();
                const transactionEncoder = getTransactionEncoder();
                if (transactions.length > 1) {
                    throw new Error("should only have one transaction");
                }
                if (transactions.length === 0) {
                    return [];
                }
                if (!connectedWalletAccount) {
                    throw new Error("no connected wallet account");
                }
                const [transaction] = transactions;
                const wireTransactionBytes =
                    transactionEncoder.encode(transaction);
                const walletAccount =
                    getWalletAccountForUiWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(
                        connectedWalletAccount,
                    );

                const inputWithOptions = {
                    ...options,
                    transaction: wireTransactionBytes as Uint8Array,
                    chain: "solana:devnet" as IdentifierString,
                    account: walletAccount,
                };
                const [{ signature }] = await getAbortablePromise(
                    signAndSendTransactionFeature.signAndSendTransaction(
                        inputWithOptions,
                    ),
                    abortSignal,
                );
                console.log("sign signature", signature);
                return Object.freeze([signature as SignatureBytes]);
            },
        };
        const message = pipe(
            createTransactionMessage({ version: 0 }),
            (m) =>
                setTransactionMessageFeePayerSigner(
                    transactionSendingSigner,
                    m,
                ),
            (m) =>
                setTransactionMessageLifetimeUsingBlockhash(latestBlockhash, m),
            (m) =>
                appendTransactionMessageInstruction(
                    getTransferSolInstruction({
                        amount: lamports(100_000_000n),
                        destination: receiver,
                        source: transactionSendingSigner,
                    }),
                    m,
                ),
        );
        const signature =
            await signAndSendTransactionMessageWithSigners(message);
        console.log("tx signature", signature);
    }
</script>

<div class="container mx-auto">
    {#if connectedWalletAccount}
        <div class="dropdown dropdown-end">
            <div tabindex="0" role="button" class="btn btn-primary m-1">
                {connectedWalletAccount.address}
            </div>
            <ul
                class="dropdown-content menu bg-base-100 rounded-box z-[1] w-52 p-2 shadow"
            >
                <li>
                    <button
                        on:click={() =>
                            copyAddress(connectedWalletAccount?.address)}
                        >Copy Address</button
                    >
                </li>
                <li>
                    <button on:click={() => disconnectWallet()}
                        >Disconnect</button
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

    {#if connectedWalletAccount}
        <div class="card bg-base-100 w-full shadow-xl">
            <div class="card-body">
                <h2 class="card-title">Transfer 0.1 SOL on devnet</h2>
                <p>receiver: FDjn87xPsLiXwakFygi4uEdet568o7A22UboxrUCwu7A</p>
                <div class="card-actions justify-end">
                    <button
                        on:click={() => handleTransfer()}
                        class="btn btn-primary">Transfer</button
                    >
                </div>
            </div>
        </div>
    {/if}
</div>
