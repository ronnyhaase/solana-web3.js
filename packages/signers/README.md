[![npm][npm-image]][npm-url]
[![npm-downloads][npm-downloads-image]][npm-url]
[![semantic-release][semantic-release-image]][semantic-release-url]
<br />
[![code-style-prettier][code-style-prettier-image]][code-style-prettier-url]

[code-style-prettier-image]: https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square
[code-style-prettier-url]: https://github.com/prettier/prettier
[npm-downloads-image]: https://img.shields.io/npm/dm/@solana/signers/experimental.svg?style=flat
[npm-image]: https://img.shields.io/npm/v/@solana/signers/experimental.svg?style=flat
[npm-url]: https://www.npmjs.com/package/@solana/signers/v/experimental
[semantic-release-image]: https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg
[semantic-release-url]: https://github.com/semantic-release/semantic-release

# @solana/signers

This package provides an abstraction layer over signing messages and transactions in Solana. It can be used standalone, but it is also exported as part of the Solana JavaScript SDK [`@solana/web3.js@experimental`](https://github.com/solana-labs/solana-web3.js/tree/master/packages/library).

You can think of signers as an abstract way to sign messages and transactions. This could be using a Crypto KeyPair, a wallet adapter in the browser, a Noop signer for testing purposes, or anything you want. Here's an example using a `CryptoKeyPair` signer:

```ts
import { pipe } from '@solana/functional';
import { generateKeyPairSigner } from '@solana/signers';
import { createTransaction } from '@solana/transactions';

// Generate a key pair signer.
const mySigner = await generateKeyPairSigner();
mySigner.address; // Address;

// Sign one or multiple messages.
const myMessage = createSignableMessage('Hello world!');
const [messageSignatures] = await mySigner.signMessages([myMessage]);

// Sign one or multiple transactions.
const myTransaction = pipe(
    createTransaction({ version: 0 })
    // Add instructions, fee payer, lifetime, etc.
);
const [transactionSignatures] = await mySigner.signTransactions([myTransaction]);
```

As you can see, this provides a consistent API regardless of how things are being signed behind the scenes. If tomorrow we need to use a browser wallet instead, we'd simply need to swap the `generateKeyPairSigner` function with the signer factory of our choice.

This package offers a total of five different types of signers that may be used in combination when applicable. Three of them allow us to sign transactions whereas the other two are used for regular message signing.

They are separated into three categories:

-   **Partial signers**: Given a message or transaction, provide one or more signatures for it. These signers are not able to modify the given data which allows us to run many of them in parallel.
-   **Modifying signers**: Can choose to modify a message or transaction before signing it with zero or more private keys. Because modifying a message or transaction invalidates any pre-existing signatures over it, modifying signers must do their work before any other signer.
-   **Sending signers**: Given a transaction, signs it and sends it immediately to the blockchain. When applicable, the signer may also decide to modify the provided transaction before signing it. This interface accommodates wallets that simply cannot sign a transaction without sending it at the same time. This category of signers does not apply to regular messages.

Thus, we end up with the following interfaces.

|                     | Partial signers            | Modifying signers            | Sending signers            |
| ------------------- | -------------------------- | ---------------------------- | -------------------------- |
| `TransactionSigner` | `TransactionPartialSigner` | `TransactionModifyingSigner` | `TransactionSendingSigner` |
| `MessageSigner`     | `MessagePartialSigner`     | `MessageModifyingSigner`     | N/A                        |

We will go through each of these five signer interfaces and their respective characteristics in the documentation below.

This package also provides the following concrete signer implementations:

-   The `KeyPairSigner` which uses a `CryptoKeyPair` to sign messages and transactions.
-   The Noop signer which does not sign anything and is mostly useful for testing purposes or for indicating that an account will be signed in a different environment (e.g. sending a transaction to your server so it can sign it).

Additionally, this package allows transaction signers to be stored inside the account meta of an instruction. This allows us to create instructions by passing around signers instead of addresses when applicable which, in turn, allows us to sign an entire transaction automatically without having to scan through its instructions to find the required signers.

In the sections below, we'll go through all the provided signers in more detail before diving into storing signers inside instruction account metas and how to benefit from it.

## Signing messages

### Types

#### `SignableMessage`

Defines a message with any of the signatures that might have already been provided by other signers. This interface allows modifying signers to decide on whether or not they should modify the provided message depending on whether or not signatures already exist for such message. It also helps create a more consistent API by providing a structure analogous to transactions which also keep track of their signature dictionary.

```ts
type SignableMessage = {
    content: Uint8Array;
    signatures: SignatureDictionary; // Record<Address, SignatureBytes>
};
```

#### `MessagePartialSigner<TAddress>`

An interface that signs an array of `SignableMessages` without modifying their content. It defines a `signMessages` function that returns a `SignatureDictionary` for each provided message. Such signature dictionaries are expected to be merged with the existing ones if any.

```ts
const myMessagePartialSigner: MessagePartialSigner<'1234..5678'> = {
    address: address('1234..5678'),
    signMessages: async (messages: SignableMessage[]): Promise<SignatureDictionary[]> => {
        // My custom signing logic.
    },
};
```

**Characteristics**:

-   **Parallel**. When multiple signers sign the same message, we can perform this operation in parallel to obtain all their signatures.
-   **Flexible order**. The order in which we use these signers for a given message doesn’t matter.

#### `MessageModifyingSigner<TAddress>`

An interface that potentially modifies the content of the provided `SignableMessages` before signing them. E.g. this enables wallets to prefix or suffix nonces to the messages they sign. For each message, instead of returning a `SignatureDirectory`, its `modifyAndSignMessages` function returns its updated `SignableMessage` with a potentially modified content and signature dictionary.

```ts
const myMessageModifyingSigner: MessageModifyingSigner<'1234..5678'> = {
    address: address('1234..5678'),
    modifyAndSignMessages: async (messages: SignableMessage[]): Promise<SignableMessage[]> => {
        // My custom signing logic.
    },
};
```

**Characteristics**:

-   **Sequential**. Contrary to partial signers, these cannot be executed in parallel as each call can modify the content of the message.
-   **First signers**. For a given message, a modifying signer must always be used before a partial signer as the former will likely modify the message and thus impact the outcome of the latter.
-   **Potential conflicts**. If more than one modifying signer is provided, the second signer may invalidate the signature of the first one. However, modifying signers may decide not to modify a message based on the existence of signatures for that message.

#### `MessageSigner<TAddress>`

Union interface that uses any of the available message signers.

```ts
type MessageSigner<TAddress extends string = string> =
    | MessagePartialSigner<TAddress>
    | MessageModifyingSigner<TAddress>;
```

### Functions

#### `createSignableMessage(content, signatures?)`

Creates a `SignableMessage` from a `Uint8Array` or a UTF-8 string. It optionally accepts a signature dictionary if the message already contains signatures.

```ts
const myMessage = createSignableMessage(new Uint8Array([1, 2, 3]));
const myMessageFromText = createSignableMessage('Hello world!');
const myMessageWithSignatures = createSignableMessage('Hello world!', {
    '1234..5678': new Uint8Array([1, 2, 3]),
});
```

#### Type guards

Each of the message interfaces described above comes with two type guards that allow us to check whether or not a given value is a message signer of the requested type. One that returns a boolean and one that asserts by throwing an error if the provided value is not of the expected interface.

```ts
const myAddress = address('1234..5678');

isMessagePartialSigner({ address: myAddress, signMessages: async () => {} }); // ✅ true
isMessagePartialSigner({ address: myAddress }); // ❌ false
assertIsMessagePartialSigner({ address: myAddress, signMessages: async () => {} }); // ✅ void
assertIsMessagePartialSigner({ address: myAddress }); // ❌ Throws an error.

isMessageModifyingSigner({ address: myAddress, modifyAndSignMessages: async () => {} }); // ✅ true
isMessageModifyingSigner({ address: myAddress }); // ❌ false
assertIsMessageModifyingSigner({ address: myAddress, modifyAndSignMessages: async () => {} }); // ✅ void
assertIsMessageModifyingSigner({ address: myAddress }); // ❌ Throws an error.

isMessageSigner({ address: myAddress, signMessages: async () => {} }); // ✅ true
isMessageSigner({ address: myAddress, modifyAndSignMessages: async () => {} }); // ✅ true
assertIsMessageSigner({ address: myAddress, signMessages: async () => {} }); // ✅ void
assertIsMessageSigner({ address: myAddress, modifyAndSignMessages: async () => {} }); // ✅ void
```

## Signing transactions

### Types

#### `TransactionPartialSigner<TAddress>`

An interface that signs an array of `CompilableTransactions` without modifying their content. It defines a `signTransactions` function that returns a `SignatureDictionary` for each provided transaction. Such signature dictionaries are expected to be merged with the existing ones if any.

```ts
const myTransactionPartialSigner: TransactionPartialSigner<'1234..5678'> = {
    address: address('1234..5678'),
    signTransactions: async (transactions: CompilableTransaction[]): Promise<SignatureDictionary[]> => {
        // My custom signing logic.
    },
};
```

**Characteristics**:

-   **Parallel**. It returns a signature directory for each provided transaction without modifying them, making it possible for multiple partial signers to sign the same transaction in parallel.
-   **Flexible order**. The order in which we use these signers for a given transaction doesn’t matter.

#### `TransactionModifyingSigner<TAddress>`

An interface that potentially modifies the provided `CompilableTransactions` before signing them. E.g. this enables wallets to inject additional instructions into the transaction before signing them. For each transaction, instead of returning a `SignatureDirectory`, its `modifyAndSignTransactions` function returns an updated `CompilableTransaction` with a potentially modified set of instructions and signature dictionary.

```ts
const myTransactionModifyingSigner: TransactionModifyingSigner<'1234..5678'> = {
    address: address('1234..5678'),
    modifyAndSignTransactions: async <T extends CompilableTransaction>(transactions: T[]): Promise<T[]> => {
        // My custom signing logic.
    },
};
```

**Characteristics**:

-   **Sequential**. Contrary to partial signers, these cannot be executed in parallel as each call can modify the provided transactions.
-   **First signers**. For a given transaction, a modifying signer must always be used before a partial signer as the former will likely modify the transaction and thus impact the outcome of the latter.
-   **Potential conflicts**. If more than one modifying signer is provided, the second signer may invalidate the signature of the first one. However, modifying signers may decide not to modify a transaction based on the existence of signatures for that transaction.

#### `TransactionSendingSigner<TAddress>`

An interface that signs one or multiple transactions before sending them immediately to the blockchain. It defines a `signAndSendTransactions` function that returns the transaction signature (i.e. its identifier) for each provided `CompilableTransaction`. This interface is required for PDA wallets and other types of wallets that don't provide an interface for signing transactions without sending them.

Note that it is also possible for such signers to modify the provided transactions before signing and sending them. This enables use cases where the modified transactions cannot be shared with the app and thus must be sent directly.

```ts
const myTransactionSendingSigner: TransactionSendingSigner<'1234..5678'> = {
    address: address('1234..5678'),
    signAndSendTransactions: async (transactions: CompilableTransaction[]): Promise<SignatureBytes[]> => {
        // My custom signing logic.
    },
};
```

**Characteristics**:

-   **Single signer**. Since this signer also sends the provided transactions, we can only use a single `TransactionSendingSigner` for a given set of transactions.
-   **Last signer**. Trivially, that signer must also be the last one used.
-   **Potential conflicts**. Since signers may decide to modify the given transactions before sending them, they may invalidate previous signatures. However, signers may decide not to modify a transaction based on the existence of signatures for that transaction.
-   **Potential confirmation**. Whilst this is not required by this interface, it is also worth noting that most wallets will also wait for the transaction to be confirmed (typically with a `confirmed` commitment) before notifying the app that they are done.

#### `TransactionSigner<TAddress>`

Union interface that uses any of the available transaction signers.

```ts
type TransactionSigner<TAddress extends string = string> =
    | TransactionPartialSigner<TAddress>
    | TransactionModifyingSigner<TAddress>
    | TransactionSendingSigner<TAddress>;
```

### Functions

#### Type guards

Each of the transaction interfaces described above comes with two type guards that allow us to check whether or not a given value is a transaction signer of the requested type. One that returns a boolean and one that asserts by throwing an error if the provided value is not of the expected interface.

```ts
const myAddress = address('1234..5678');

isTransactionPartialSigner({ address: myAddress, signTransactions: async () => {} }); // ✅ true
isTransactionPartialSigner({ address: myAddress }); // ❌ false
assertIsTransactionPartialSigner({ address: myAddress, signTransactions: async () => {} }); // ✅ void
assertIsTransactionPartialSigner({ address: myAddress }); // ❌ Throws an error.

isTransactionModifyingSigner({ address: myAddress, modifyAndSignTransactions: async () => {} }); // ✅ true
isTransactionModifyingSigner({ address: myAddress }); // ❌ false
assertIsTransactionModifyingSigner({ address: myAddress, modifyAndSignTransactions: async () => {} }); // ✅ void
assertIsTransactionModifyingSigner({ address: myAddress }); // ❌ Throws an error.

isTransactionSendingSigner({ address: myAddress, signAndSignTransaction: async () => {} }); // ✅ true
isTransactionSendingSigner({ address: myAddress }); // ❌ false
assertIsTransactionSendingSigner({ address: myAddress, signAndSignTransaction: async () => {} }); // ✅ void
assertIsTransactionSendingSigner({ address: myAddress }); // ❌ Throws an error.

isTransactionSigner({ address: myAddress, signTransactions: async () => {} }); // ✅ true
isTransactionSigner({ address: myAddress, modifyAndSignTransactions: async () => {} }); // ✅ true
isTransactionSigner({ address: myAddress, signAndSignTransaction: async () => {} }); // ✅ true
assertIsTransactionSigner({ address: myAddress, signTransactions: async () => {} }); // ✅ void
assertIsTransactionSigner({ address: myAddress, modifyAndSignTransactions: async () => {} }); // ✅ void
assertIsTransactionSigner({ address: myAddress, signAndSignTransaction: async () => {} }); // ✅ void
```

## Creating and generating KeyPair signers

### Types

#### `KeyPairSigner<TAddress>`

Defines a signer that uses a `CryptoKeyPair` to sign messages and transactions. It implements both the `MessagePartialSigner` and `TransactionPartialSigner` interfaces and keeps track of the `CryptoKeyPair` instance used to sign messages and transactions.

```ts
import { generateKeyPairSigner } from '@solana/signers';

const myKeyPairSigner = generateKeyPairSigner();
myKeyPairSigner.address; // Address;
myKeyPairSigner.keyPair; // CryptoKeyPair;
const [myMessageSignatures] = await myKeyPairSigner.signMessages([myMessage]);
const [myTransactionSignatures] = await myKeyPairSigner.signTransactions([myTransaction]);
```

### Functions

#### `createSignerFromKeyPair()`

Creates a `KeyPairSigner` from a provided Crypto KeyPair. The `signMessages` and `signTransactions` functions of the returned signer will use the private key of the provided key pair to sign messages and transactions. Note that both the `signMessages` and `signTransactions` implementations are parallelized, meaning that they will sign all provided messages and transactions in parallel.

```ts
import { generateKeyPair } from '@solana/keys';
import { createSignerFromKeyPair, KeyPairSigner } from '@solana/signers';

const myKeyPair: CryptoKeyPair = await generateKeyPair();
const myKeyPairSigner: KeyPairSigner = await createSignerFromKeyPair(myKeyPair);
```

#### `generateKeyPairSigner()`

A convenience function that generates a new Crypto KeyPair and immediately creates a `KeyPairSigner` from it.

```ts
import { generateKeyPairSigner } from '@solana/signers';

const myKeyPairSigner = await generateKeyPairSigner();
```

#### `isKeyPairSigner()`

A type guard that returns `true` if the provided value is a `KeyPairSigner`.

```ts
const myKeyPairSigner = await generateKeyPairSigner();
isKeyPairSigner(myKeyPairSigner); // ✅ true
isKeyPairSigner({ address: address('1234..5678') }); // ❌ false
```

#### `assertIsKeyPairSigner()`

A type guard that throws an error if the provided value is not a `KeyPairSigner`.

```ts
const myKeyPairSigner = await generateKeyPairSigner();
assertIsKeyPairSigner(myKeyPairSigner); // ✅ void
assertIsKeyPairSigner({ address: address('1234..5678') }); // ❌ Throws an error.
```

## Creating Noop signers

### Functions

#### `createNoopSigner()`

_Coming soon..._

Creates a Noop (No-Operation) signer from a given address. It will return an implementation of both the `MessagePartialSigner` and `TransactionPartialSigner` interfaces that do not sign anything. Namely, signing a transaction or a message will return an empty `SignatureDictionary`.

This signer may be useful:

-   For testing purposes.
-   For indicating that a given account is a signer and taking the responsibility to provide the signature for that account ourselves. For instance, if we need to send the transaction to a server that will sign it and send it for us.

```ts
import { createNoopSigner } from '@solana/signers';

const myAddress = address('1234..5678');
const myNoopSigner = createNoopSigner(myAddress);
// ^ MessagePartialSigner<'1234..5678'> & TransactionPartialSigner<'1234..5678'>
```

## Storing transaction signers inside instruction account metas

This package defines an alternative definition for account metas that allows us to store `TransactionSigners` inside them. This means each instruction can keep track of its own set of signers and, by extension, so can transactions.

It also provides helper functions that deduplicate and extract signers from instructions and transactions which makes it possible to sign an entire transaction automatically as we will see in the next section.

### Types

#### `IAccountSignerMeta`

Alternative `IAccountMeta` definition for signer accounts that allows us to store `TransactionSigners` inside it.

```ts
const mySignerMeta: IAccountSignerMeta = {
    address: myTransactionSigner.address,
    role: AccountRole.READONLY_SIGNER,
    signer: myTransactionSigner,
};
```

#### `IInstructionWithSigners`

Extends the `IInstruction` type to allow `IAccountSignerMetas` to be used inside the instruction's `accounts` array.

```ts
const myInstructionWithSigners: IInstructionWithSigners = {
    programAddress: address('1234..5678'),
    accounts: [
        {
            address: myTransactionSigner.address,
            role: AccountRole.READONLY_SIGNER,
            signer: myTransactionSigner,
        },
    ],
};
```

### Functions

#### `getSignersFromInstruction()`

Extracts and deduplicates all signers stored inside the account metas of an instruction.

```ts
const mySignerA = { address: address('1111..1111'), signTransactions: async () => {} };
const mySignerB = { address: address('2222..2222'), signTransactions: async () => {} };
const myInstructionWithSigners: IInstructionWithSigners = {
    programAddress: address('1234..5678'),
    accounts: [
        { address: mySignerA.address, role: AccountRole.READONLY_SIGNER, signer: mySignerA },
        { address: mySignerB.address, role: AccountRole.WRITABLE_SIGNER, signer: mySignerB },
        { address: mySignerA.address, role: AccountRole.WRITABLE_SIGNER, signer: mySignerA },
    ],
};

const instructionSigners = getSignersFromInstruction(myInstructionWithSigners);
// ^ [mySignerA, mySignerB]
```

#### `getSignersFromTransaction()`

Similarly to `getSignersFromInstruction`, this function extracts and deduplicates all signers stored inside the account metas of all the instructions inside a transaction.

```ts
const transactionSigners = getSignersFromTransaction(myTransactionWithSigners);
```

## Signing transactions with signers

As we've seen in the previous section, we can store and extract `TransactionSigners` from instructions and transactions. This allows us to provide helper methods that sign transactions using the signers stored inside them.

### Functions

#### `partiallySignTransactionWithSigners()`

Extracts all signers inside the provided transaction and uses them to sign it. It first uses all `TransactionModifyingSigners` sequentially before using all `TransactionPartialSigners` in parallel.

If a composite signer implements both interfaces, it will be used as a modifying signer if no other signer implements that interface. Otherwise, it will be used as a partial signer.

```ts
const mySignedTransaction = partiallySignTransactionWithSigners(myTransaction);
```

It also accepts an additional array of signers that will be merged with the ones extracted from the transaction, if any.

```ts
const mySignedTransaction = partiallySignTransactionWithSigners(myTransaction, [myOtherSigner]);
```

Finally, note that this function ignores `TransactionSendingSigners` as it does not send the transaction. See the `signAndSendTransactionWithSigners` function below for more details on how to use sending signers.

#### `signTransactionWithSigners()`

This function works the same as the `partiallySignTransactionWithSigners` function described above except that it also ensures the transaction is fully signed before returning it. An error will be thrown if that's not the case.

```ts
const mySignedTransaction = signTransactionWithSigners(myTransaction);

// With additional signers.
const mySignedTransaction = signTransactionWithSigners(myTransaction, [myOtherSigner]);

// We now know the transaction is fully signed.
mySignedTransaction satisfies IFullySignedTransaction;
```

#### `signAndSendTransactionWithSigners()`

Extracts all signers inside the provided transaction and uses them to sign it before sending it immediately to the blockchain. It returns the signature of the sent transaction (i.e. its identifier).

```ts
const myTransactionSignature = signAndSendTransactionWithSigners(myTransaction);

// With additional signers.
const myTransactionSignature = signAndSendTransactionWithSigners(myTransaction, [myOtherSigner]);
```

Similarly to the `partiallySignTransactionWithSigners` function, it first uses all `TransactionModifyingSigners` sequentially before using all `TransactionPartialSigners` in parallel. It then sends the transaction using the first `TransactionSendingSigner` it finds. Any other sending signer that does not implement another transaction signer interface will be ignored.

If no `TransactionSendingSigner` is extracted from the transaction or explicitly provided, the `fallbackSender` third argument will be used to send the transaction. If no fallback sender is provided and no sending signer is identified, an error will be thrown.

```ts
const fallbackSender = async (transaction: CompilableTransaction) => {
    const encodedTransaction = getBase64EncodedWireTransaction(transaction);
    const signature = await rpc.sendTransaction(encodedTransaction).send();
    return getBase58Encoder().encode(signature);
};

const myTransactionSignature = signAndSendTransactionWithSigners(myTransaction, [], fallbackSender);
```

Here as well, composite transaction signers are treated such that at least one sending signer is used if any. When a `TransactionSigner` implements more than one interface, use it as a:

-   `TransactionSendingSigner`, if no other `TransactionSendingSigner` exist.
-   `TransactionModifyingSigner`, if no other `TransactionModifyingSigner` exist.
-   `TransactionPartialSigner`, otherwise.