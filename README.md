![banner](https://github.com/thirdweb-example/thirdweb-siwf/assets/17715009/9ec0129f-65ff-4016-a6a0-1e5a4efdad77)

# thirdweb Sign in with Farcaster Example

[<img alt="thirdweb SDK" src="https://img.shields.io/npm/v/thirdweb?label=Thirdweb SDK&style=for-the-badge&logo=npm" height="30">](https://www.npmjs.com/package/thirdweb)
[<img alt="Discord" src="https://img.shields.io/discord/834227967404146718.svg?color=7289da&label=discord&logo=discord&style=for-the-badge" height="30">](https://discord.gg/thirdweb)

This example app showcases how to authenticate users via Farcaster (specifically Warpcast) and generate a unique reusable smart wallet from their Farcaster signature.

## Getting Started

> This project assumes some basic knowledge of TypeScript, Next.js App Router, and [Connect SDK](https://portal.thirdweb.com/typescript/v5).

## Environment Variables

1. Create your `.env` by running `cp .env.example .env` in the project root.

2. Create a client ID from the [thirdweb dashboard](https://thirdweb.com/dashboard/settings/api-keys) and add it to your `.env` as `NEXT_PUBLIC_THIRDWEB_CLIENT_ID`.

3. Deploy an [`AccountFactory`](https://thirdweb.com/thirdweb.eth/AccountFactory) on your chain of choice from the thirdweb dashboard and paste the contract address in your `.env` as `NEXT_PUBLIC_FACTORY_ADDRESS`.

4. Deploy a new [`OpenEditionERC721`](https://thirdweb.com/thirdweb.eth/OpenEditionERC721) contract on your chain of choice from the thirdweb dashboard and add a claim period with no restrictions. Paste the contract's address in your `.env` as `NEXT_PUBLIC_NFT_ADDRESS`.

5. Set your chain ID, block explorer base url (this should be the base url up until the transaction hash, for etherscan mainnet it would be `https://etherscan.io/tx/`), and an encryption key of your choice in your `.env`

## Set authentication endpoint

This project uses an incredibly powerful thirdweb feature called [Authentication Endpoints](https://portal.thirdweb.com/connect/in-app-wallet/custom-auth/custom-auth-server). It uses your own API endpoint to generate a wallet for users on successful authentication. All the code for this is written for you in this project, you'll just need to set the endpoint in your thirdweb dashboard.

> To use Custom Authentication Endpoints, you'll need to be on the Growth Plan. If you have questions about the plan options or want to try it out, [reach out to our team](https://thirdweb.com/contact-us).

Navigate to the [In-App Wallets](https://thirdweb.com/dashboard/connect/in-app-wallets) page on the dashboard and select your project from the dropdown. **This should be the same project your `clientId` is from.** Then click the **"Configuration" tab** and scroll down to "Custom Authentication Endpoint" and enable the toggle. You'll then see a field to enter your endpoint.

<img width="1215" alt="Screenshot 2024-04-25 at 10 46 11 PM" src="https://github.com/thirdweb-example/thirdweb-siwf/assets/17715009/b05825c1-96b3-4e58-908e-c47002c4c02f">

While testing the project locally, you'll need a publicly exposed endpoint to authenticate through. We recommend using a tool like [ngrok](https://ngrok.com/product/secure-tunnels) to create a public endpoint that forwards traffic to your local server. Forward your traffic to `http://localhost:3000` (where your app will run locally).

Once you have your ngrok or similar endpoint, add it to the Authentication Endpoint field as `[YOUR FORWARDING ENDPOINT]/api/authenticate`, the route this app uses to perform authentication.

You're now ready to run the project!

> **When you deploy to production (or any live URL), you'll modify this authentication endpoint to be your actual live URL. You could also create a separate thirdweb project for local development and production.**

### Run the project

You're now ready to test the project! First, install the dependencies:

```bash
pnpm install
```

Then, run the app locally:

```bash
pnpm run dev
```

You should see the app at http://localhost:3000. Try signing in with Warpcast and minting the NFT!

Check the users tab in [In-App Wallets](https://thirdweb.com/dashboard/connect/in-app-wallets) dashboard. You should see your created users appear.

### Going to production

Once you've implemented this flow into your own app, there are a few changes you'll need to make to go to production.

1. Modify the `NEXT_PUBLIC_DOMAIN` in your production `.env` to be your production domain.
    > Don't prepend your domain with https:// when setting it in your `.env`
2. Remember to go to your project in the [In-App Wallets](https://thirdweb.com/dashboard/connect/in-app-wallets) configuration tab and update the auth endpoint to be `[YOUR PRODUCTION URL]/api/authenticate`. In this case, do include `https://` in the URL.

You might also want to (but don't have to) use a different account factory, NFT, and chain for production.

Now, you're ready to deploy your app to Vercel or a similar service.

## How it works

All the logic for this example can be found in `page.tsx`. The most important areas for authentication are `handleSuccess()`, `mint`, and the `useConnect` hook from the thirdweb React SDK.

### Authenticating the user

We use [Farcaster AuthKit](https://docs.farcaster.xyz/auth-kit/introduction) to handle the connection with Warpcast. We use the `useSignIn` hook to trigger the `handleSuccess` as a success callback when the sign in data is available. When the user authenticates with Warpcast, this function will be called with the user's signature.

In `handleSuccess`, we optimistically set the fid (it will unset if the signature verification or wallet generation fail), then connect using the `"auth_endpoint"` strategy. This strategy needs a `payload` and `encryptionKey` that will be sent to the authentication endpoint we specified in the dashboard. The code below is simplified from the actual project.

```ts
const handleSuccess = async (res: StatusAPIResponse) => {
	await wallet.connect({
		client: thirdwebClient,
		chain: defineChain(Number(process.env.NEXT_PUBLIC_CHAIN_ID)),
		strategy: "auth_endpoint",
		payload: JSON.stringify({
			signature: res.signature,
			message: res.message,
			nonce: res.nonce,
		}),
		encryptionKey: process.env.NEXT_PUBLIC_ENCRYPTION_KEY!,
	});
	await connect(wallet);
};
```

Then in the `/api/authenticate/route.ts` file we specify a `POST` handler that accepts the payload and verifies the signature. If this route returns a `userId`, it's considered to be successful and generates and/or connects the user's in-app wallet.

### Using a smart wallet

Since this user's wallet is generated the first time they sign into our app, it won't have any funds for gas. Instead, we'll wrap this generated wallet in a smart wallet, which will allow the user to execute gasless transactions.

We use the `useConnect` hook from the [React SDK](https://portal.thirdweb.com/typescript/v5/react) to specify the client and account abstraction options (gasless enabled, factory address, and chain). This hook returns a `connect` function that will wrap our in-app wallet and set the app's currently active wallet to this smart wallet.

```ts
const { connect } = useConnect({
	client: thirdwebClient,
	accountAbstraction: {
		gasless: true,
		chain: defineChain(Number(process.env.NEXT_PUBLIC_CHAIN_ID)),
		factoryAddress: process.env.NEXT_PUBLIC_FACTORY_ADDRESS as Address,
	},
});
```

> Note; We setup a [ThirdwebProvider](https://portal.thirdweb.com/typescript/v5/react/ThirdwebProvider) in `Providers.tsx` for the `useConnect` and `useActiveAccount` hooks to work.

Once our smart wallet is connected, the `useActiveAccount` hook will return it, allowing us to enable minting the NFT.

### Minting the NFT

Once the smart account is ready we enable the minting button. When clicked, it calls `mint`, a simple function that uses the Thirdweb SDK's ERC721 extension to generate a `claimTo` transaction, then we send and await the transaction result in one `sendAndConfirmTransaction` call. With extensions, we don't need to worry about ABIs, argument arrays, calldata, or any other "low-level" concepts. All the complicated elements are abstracted away from the frontend code.

```ts
async function mint(account: Account, recipient: Address) {
	const contract = getContract({
		address: process.env.NEXT_PUBLIC_NFT_ADDRESS as Address,
		chain: defineChain(Number(process.env.NEXT_PUBLIC_CHAIN_ID)),
		client: thirdwebClient,
	});

	const mintTx = claimTo({
		contract,
		to: recipient,
		quantity: BigInt(1),
	});

	const res = await sendAndConfirmTransaction({
		account,
		transaction: mintTx,
	});

	return res.transactionHash;
}
```

## Documentation

-   [TypeScript SDK](https://portal.thirdweb.com/typescript/v5)
-   [Next.js Docs](https://nextjs.org/docs)

## Support

For help or feedback, please [visit our support site](https://thirdweb.com/support)
