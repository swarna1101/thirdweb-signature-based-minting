# Signature Based Minting with Next.JS

## Introduction

we will utilize signature-based minting of NFTs to create a "community-built" NFT collection, where each user mints and owns their own NFT based on specific conditions that we provide.

## Tools:

- [**thirdweb React SDK**](https://docs.thirdweb.com/react): to enable users to connect and disconnect their wallets to our website, using [useMetamask](https://docs.thirdweb.com/react/react.usemetamask) & [useDisconnect](https://docs.thirdweb.com/react/react.usedisconnect), and prompt them to approve transactions with MetaMask. We'll also use the helpful hooks such as [useNFTs](https://docs.thirdweb.com/react/react.usenfts) to easily fetch data from the smart contract.
- [**thirdweb NFT Collection**](https://portal.thirdweb.com/contracts/nft-collection): to create an ERC721 NFT Collection that our community can mint NFTs into.
- [**thirdweb TypeScript SDK**](https://docs.thirdweb.com/typescript): to connect to our NFT Collection Smart contract via React hooks such as [useNFTCollection](https://docs.thirdweb.com/react/react.usenftcollection), mint new NFTs with [signature based minting](https://docs.thirdweb.com/typescript/sdk.nftcollection.signature).
- [**Next JS API Routes**](https://nextjs.org/docs/api-routes/introduction): For us to securely generate signatures on the server-side, on behalf of our wallet, using our wallet's private key.

## What is Signature Based Minting?

NFT Collections come in different shapes, sizes, and rules.

On one end of the spectrum, an NFT collection can be a set amount of NFTs, and they're all made by one wallet, once they're all claimed/minted, no more NFTs are ever minted in the collection. A popular example of this is the [Bored Ape Yacht Club Collection](https://boredapeyachtclub.com/#/).

On the other end of the spectrum, an NFT collection could also start out with no NFTs! NFTs in _this_ collection could be created by anyone, at any time, and the NFTs could look completely different. This type of collection is a bit less well-known, but demonstrates that NFT collections can be totally different!

Signature-based minting enables a use case that is somewhere between these two ends of the spectrum. What if you didn't want **EVERYONE** to be able to mint an NFT into your collection, or you only wanted specific types of NFTs to be minted?

Signature-based minting allows you to specify exactly what NFTs you allow to be minted into your NFT collection, by generating a signature for an NFT. This signature can be used by a wallet address that you specify to mint that NFT.

In this guide, we'll grant users signatures to mint NFTs into our collection that follow the format we expect.

In the end, we'll have a community-made collection that only contains NFTs with animal names, by restricting the signatures we provide to only allow NFTs with one of our [animal names](./animalNames.ts)!

Let's get into it!

## Setting Up The NFT Collection

To create an NFT collection, we can use the thirdweb dashboard and create a fully customizable NFT collection in just a few clicks.

Head to https://thirdweb.com/dashboard and create a new contract on the Polygon Mumbai network. Click **Create NFTs and Tokens** > **NFT Collection**.

Give your collection a name and click **Deploy now**!

Nice! Now we have an NFT collection. Let's set up a project and see how we can start minting some NFTs using the thirdweb SDK.

## Setting Up The Project

To get started, we have a ready-made template that includes all the code you need to work with Thirdweb, TypeScript and Next JS available here https://github.com/thirdweb-example/next-typescript-starter

Our application is wrapped in a Thirdweb Provider so that we can access Thirdweb anywhere in our application:

```tsx
import { ThirdwebProvider } from "@thirdweb-dev/react";

// This is the chain your dApp will work on.
const activeChain = "goerli";

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <ThirdwebProvider activeChain={activeChain}>
      <Component {...pageProps} />
    </ThirdwebProvider>
  );
}
```

## Connecting User's Wallets

Over at the home page at `index.tsx`, we're using the [thirdweb React SDK MetaMask Connector](https://docs.thirdweb.com/react/category/wallet-connection) so our user can connect their wallet to our website.

```tsx
import {
  useAddress,
  useDisconnect,
  useMetamask,
  useNFTCollection,
} from "@thirdweb-dev/react";

  const address = useAddress(); // Grab the current user's address
  const connectWithMetamask = useMetamask(); // Connect with metamask
  const disconnectWallet = useDisconnect(); // Disconnect from metamask

};
```

We've attached these `connectWithMetamask` and `disconnectWallet` functions to the `onClick` to fire when the user clicks the `Connect` or `Disconnect` buttons.

You can see an example of how to implement that logic in our [index.tsx file](./pages/index.tsx)

## Loading & Displaying NFTs

For our community collection, each NFT is minted by a different user. We're using another thirdweb hook called `useNFTCollection` to connect to our smart contract and load all of the NFT's in our collection.

```tsx
// Fetch the NFT collection from thirdweb via it's contract address.
const nftCollection = useNFTCollection(
  // Replace this with your NFT Collection contract address
  process.env.NEXT_PUBLIC_NFT_COLLECTION_ADDRESS
);
```

Once we've loaded them from the smart contract, we can display each NFT to the user.

We'll be using the React SDK's `useNFTs` to fetch all of the NFTs in our collection.

**Fetch All NFTS**:

```tsx
const { data: nfts, isLoading: loadingNfts } = useNFTs(nftCollection);
```

Now we've got the NFT's loaded, we can display them to the user!

We'll let you decide how best to display your NFTs, but if you're looking for an example, check out the [index.tsx file](./pages/index.tsx) to see how we did it.

We can also see who is the owner of each nft with `.owner`, and the `metadata.name` and `metadata.description` inside of each NFT!

## Creating NFTs with Signature Based Minting

The way that our signature-based minting process works is in 3 steps:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1650958559249/S8mlZIQZm.png)

1. The connected wallet calls a [Next JS API Route](https://nextjs.org/docs/api-routes/introduction) with three parameters, `authorAddress`, `nftName`, and `imagePath` inside the request body.

2. The API route runs some checks on the server-side to see if this user and the NFT they are requesting to mint is eligible or not. If the request _is_ eligible, it generates a signature to mint an NFT with a specific set of characteristics.

3. Once the API function is done processing, it sends the client/user a signature. The user can call `mint` with this signature to create an NFT with the conditions provided from our server.

The main benefit here is that we now have an environment where we (the owner of the smart contract) can decide what the user can mint into our collection, all programmatically.

## API Route

To create an API Route, you'll need to create a file in the `/pages/api` directory of your project.

On the server-side API route, we can:

- Run a few checks to see if the requested NFT meets our criteria
- Generate a signature to mint the NFT if it does.
- Send the signature back to the client/user if the NFT is eligible.

**De-structure the arguments we passed in out of the request body**:

```tsx
const { authorAddress, nftName, imagePath } = JSON.parse(req.body);
```

**Initialize the Thirdweb SDK on the server-side**

```tsx
// Initialize the Thirdweb SDK on the serverside
const sdk = ThirdwebSDK.fromPrivateKey(
  // Your wallet private key (read it in from .env.local file)
  process.env.PRIVATE_KEY as string,
  "mumbai"
);
```