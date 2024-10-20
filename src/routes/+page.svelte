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
        getWalletAccountFeature,
        type UiWalletAccount,
        type UiWallet,
    } from "@wallet-standard/ui";
    import { getTransactionEncoder } from '@solana/transactions';
    import { getAbortablePromise } from '@solana/promises';
    import { type SignatureBytes } from '@solana/keys';
    // type SignatureBytes = Uint8Array & { readonly __brand: unique symbol };

    import {
        StandardConnect,
        StandardDisconnect,
        StandardEvents,
        type StandardConnectFeature,
        type StandardDisconnectFeature,
        type StandardEventsFeature,
    } from "@wallet-standard/features";
    import type { IdentifierString, WalletAccount, Wallet, WalletWithFeatures } from "@wallet-standard/base";
    import {
        address,
        appendTransactionMessageInstruction,
        assertIsTransactionMessageWithSingleSendingSigner,
        createTransactionMessage,
        getBase58Decoder,
        lamports,
        pipe,
        setTransactionMessageFeePayerSigner,
        setTransactionMessageFeePayer,
        setTransactionMessageLifetimeUsingBlockhash,
        signAndSendTransactionMessageWithSigners,
        createSolanaRpc, createSolanaRpcSubscriptions, devnet,
        createNoopSigner,
        compileTransaction,
    } from "@solana/web3.js";

    import { getTransferSolInstruction } from "@solana-program/system";

    // defined in: https://github.com/anza-xyz/wallet-standard/blob/master/packages/core/features/src/signAndSendTransaction.ts#L10C1-L11C1
    const SolanaSignAndSendTransaction = 'solana:signAndSendTransaction';

    type SolanaSignAndSendTransactionVersion = '1.0.0';

    type SolanaTransactionVersion = 'legacy' | 0;

    type SolanaTransactionCommitment = 'processed' | 'confirmed' | 'finalized';

    type SolanaSignTransactionOptions = {
        /** Preflight commitment level. */
        readonly preflightCommitment?: SolanaTransactionCommitment;

        /** The minimum slot that the request can be evaluated at. */
        readonly minContextSlot?: number;
    };

    interface SolanaSignTransactionInput {
        /** Account to use. */
        readonly account: WalletAccount;

        /** Serialized transaction, as raw bytes. */
        readonly transaction: Uint8Array;

        /** Chain to use. */
        readonly chain?: IdentifierString;

        /** TODO: docs */
        readonly options?: SolanaSignTransactionOptions;
    }

    type SolanaSignAndSendTransactionMode = 'parallel' | 'serial';

    type SolanaSignAndSendTransactionOptions = SolanaSignTransactionOptions & {
        /** Mode for signing and sending transactions. */
        readonly mode?: SolanaSignAndSendTransactionMode;

        /** Desired commitment level. If provided, confirm the transaction after sending. */
        readonly commitment?: SolanaTransactionCommitment;

        /** Disable transaction verification at the RPC. */
        readonly skipPreflight?: boolean;

        /** Maximum number of times for the RPC node to retry sending the transaction to the leader. */
        readonly maxRetries?: number;
    };


    interface SolanaSignAndSendTransactionInput extends SolanaSignTransactionInput {
        /** Chain to use. */
        readonly chain: IdentifierString;

        /** TODO: docs */
        readonly options?: SolanaSignAndSendTransactionOptions;
    }

    interface SolanaSignAndSendTransactionOutput {
        /** Transaction signature, as raw bytes. */
        readonly signature: Uint8Array;
    }

    type SolanaSignAndSendTransactionMethod = (
        ...inputs: readonly SolanaSignAndSendTransactionInput[]
    ) => Promise<readonly SolanaSignAndSendTransactionOutput[]>;

    type SolanaSignAndSendTransactionFeature = {
        /** Name of the feature. */
        readonly [SolanaSignAndSendTransaction]: {
            /** Version of the feature API. */
            readonly version: SolanaSignAndSendTransactionVersion;

            /** TODO: docs */
            readonly supportedTransactionVersions: readonly SolanaTransactionVersion[];

            /**
             * Sign transactions using the account's secret key and send them to the chain.
             *
             * @param inputs Inputs for signing and sending transactions.
             *
             * @return Outputs of signing and sending transactions.
             */
            readonly signAndSendTransaction: SolanaSignAndSendTransactionMethod;
        };
    };

    let account: UiWalletAccount | null = null;

    // let installedUiWallets: UiWallet[] = [];

    onMount(async () => {
        // get connected account
        console.log('onmount');
        await initSolanaWallets();
    });

    const { get, on } = getWallets();

    $: outputWallets = get();

    // let installedUiWallets: UiWallet[] = [];
    $: installedUiWallets = outputWallets.map(wallet => getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(wallet));

    const walletsToChangeListenerDisposeFn = new Map<Wallet, () => void>();

    $: disposeRegisterListener = on('register', (...wallets) => {
        wallets.filter(walletHasStandardEventsFeature).map(subscribeToWalletEvents);
    });

    $: disposeUnregisterListener = on('unregister', (...wallets) => {
        wallets.forEach((wallet) => {
            const dispose = walletsToChangeListenerDisposeFn.get(wallet);
            if (!dispose) {
                // Not all wallets will have a corresponding dispose function because they
                // might not support `standard:events`.
                return;
            }
            walletsToChangeListenerDisposeFn.delete(wallet);
            dispose();
        });
    });

    $: {
        disposeRegisterListener();
        disposeUnregisterListener();
        walletsToChangeListenerDisposeFn.forEach((dispose) => dispose());
        walletsToChangeListenerDisposeFn.clear();
    }

    function walletHasStandardEventsFeature(wallet: Wallet): wallet is WalletWithFeatures<StandardEventsFeature> {
        return StandardEvents in wallet.features;
    }
    function subscribeToWalletEvents(wallet: WalletWithFeatures<StandardEventsFeature>): () => void {
        const dispose = wallet.features[StandardEvents].on('change', () => {
            outputWallets = get();
        });
        walletsToChangeListenerDisposeFn.set(wallet, dispose);
        return dispose;
    }

    

    function copyAddress(address?: string) {
        if (address ) navigator.clipboard.writeText(address);
    }
    async function initSolanaWallets() {
        // if (installedUiWallets.length > 0) return;
        for (const wallet of outputWallets) {
            const uiWallet =
                getOrCreateUiWalletForStandardWallet_DO_NOT_USE_OR_YOU_WILL_BE_FIRED(
                    wallet,
                );
            // const supportedUiWalletFeatures = getWalletFeature(uiWallet, StandardConnect);
            if (!uiWallet.chains.includes("solana:devnet")) {
                continue; // skip sui wallet or others
            }

            // switch account in browser wallet plugin will trigger this event
            // wallet.features[StandardEvents].on("change", ({ accounts }) => {
            //     console.log('on change', accounts, typeof accounts[0]);
            //     account = accounts[0];
            //     // account = getOrCreateUiWalletAccountForStandardWalletAccount_DO_NOT_USE_OR_YOU_WILL_BE_FIRED.bind(
            //     //     null,
            //     //     accounts[0]
            //     // );
            //     console.log('account', account, typeof account);
            //     // const newWallet = {...installedUiWallets[0], accounts}
            //     // // installedUiWallets[0].accounts = accounts;
            //     // installedUiWallets = [newWallet];
            // });

            // console.log("solana uiWallet", uiWallet); // not autoConnect so first load will not have account
            // // await connectWallet(uiWallet);
            // installedUiWallets = [...installedUiWallets, uiWallet];
            // if (uiWallet.accounts.length > 0) {
            //     account = uiWallet.accounts[0];
            //     break;
            // }
        }
    }

    let connectedWallet: UiWallet | null = null;

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
        connectedWallet = wallet;
    }

    async function disconnectWallet() {
        if (!connectedWallet) return;
        const disconnectFeature = getWalletFeature(
            connectedWallet,
            StandardDisconnect,
        ) as StandardDisconnectFeature[typeof StandardDisconnect];
        await disconnectFeature.disconnect();
        account = null;
    }

    async function handleTransfer() {
        // ensure account exists
        console.log('handle transfer', account);
        if (!account) return;

        const receiver = address('FDjn87xPsLiXwakFygi4uEdet568o7A22UboxrUCwu7A');

        const rpc = createSolanaRpc(devnet('https://api.devnet.solana.com'));
        const rpcSubscriptions = createSolanaRpcSubscriptions(devnet('wss://api.devnet.solana.com'));

        const { value: latestBlockhash } = await rpc
            .getLatestBlockhash({ commitment: 'confirmed' })
            .send();


        console.log('uiWallets & account', installedUiWallets, account);
        const signAndSendTransactionFeature = getWalletAccountFeature(
            account,
            SolanaSignAndSendTransaction,
        ) as SolanaSignAndSendTransactionFeature[typeof SolanaSignAndSendTransaction];
        console.log('signAndSendTransactionFeature', signAndSendTransactionFeature);

        const transactionSendingSigner = {
            address: address(account.address),
            async signAndSendTransactions(transactions, config = {}) {
                const { abortSignal, ...options } = config;
                abortSignal?.throwIfAborted();
                const transactionEncoder = getTransactionEncoder();
                if (transactions.length > 1) {
                    throw new Error('should only have one transaction');
                }
                if (transactions.length === 0) {
                    return [];
                }
                const [transaction] = transactions;
                const wireTransactionBytes = transactionEncoder.encode(transaction);
                const inputWithOptions = {
                    ...options,
                    transaction: wireTransactionBytes as Uint8Array,
                    chain: 'solana:devnet' as IdentifierString,
                    account: account,
                };
                console.log('transaction', transaction);
                const [{ signature }] = await getAbortablePromise(signAndSendTransactionFeature.signAndSendTransaction(inputWithOptions), abortSignal);
                console.log('signature', signature);
                return Object.freeze([signature as SignatureBytes]);
            },
        };
        const message = pipe(
            createTransactionMessage({ version: 0 }),
            m => setTransactionMessageFeePayerSigner(transactionSendingSigner, m),
            m => setTransactionMessageLifetimeUsingBlockhash(latestBlockhash, m),
            m =>
                appendTransactionMessageInstruction(
                    getTransferSolInstruction({
                        amount: lamports(100_000_000n),
                        destination: receiver,
                        source: transactionSendingSigner,
                    }),
                    m,
                ),
        );
        // const myTx = compileTransaction(message);

        // 为啥会报错呢
        const signature = await signAndSendTransactionMessageWithSigners(message);
        // const signature = await signAndSendTransactionFeature.signAndSendTransaction({chain: 'solana:devnet', account, transaction: new Uint8Array(myTx.messageBytes)});
        console.log("signature", signature);
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

    {#if account}
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
