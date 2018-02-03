# Stellar Stripe Anchor
This is a all-in-one package for running a USD anchor on stellar using Stripe to accept deposits and pay withdrawals. All you need is an activated Stripe account in the USA, a database, and to be able host this package on an HTTPS domain. It will run a Stellar Bridge for you, and operate entirely autonomously as long as it has sufficient balance.

Obviously this is opinionated, but is a great base for your own bridge application. Alternatively, it will run out of the box without you needing to learn Stripe or Horizon APIs or worry about edge cases.

## Dependencies
For [bridge server](https://github.com/stellar/bridge-server):

- Go
- A running MySQL or PostgreSQL database

For anchor webserver:

- node.js
- yarn
- A domain with HTTPS (to go into production)

## Setup
### Installing dependencies

1. Put your SQL database details in `bridge.cfg`
2. `yarn setup`

This will install node modules, do a first build of the frontend, pull in the bridge-server submodule and build it, and setup your database.

### Stripe dashboard setup

1. Sign up and activate your account. Your location must be set to United States.
2. In Radar - Rules, ensure you block payments if CVC or ZIP code verification fails. This will protect you from fraud.
3. In Balance - Settings, set payouts to manual, as you need to keep balance on the platform in case of withdrawals.
2. Go to Connect - Settings and input branding details (name, logo, domain). Also add a Redirect URI. In production the default should be `https://yourdomain.com` but for testing you should set the default to `http://localhost:3000`. Take note of your client ID.
3. There are three keys you need from Stripe to configure this application, your publishable API key, your secret key, which are in the API tab, and your Connect client ID. You can get these seperately for Stripe test mode as well (use the test mode slider on the left of the Stripe dashboard)

## Configuration
The key areas where you will need to change variables are:

| File                                    | Configured variables                                                                  | Purpose                          |
|-----------------------------------------|---------------------------------------------------------------------------------------|----------------------------------|
| `serverConfig.js`                       | Stripe secret and Connect ID, issuer account ID and seed,  server port.                                | API server                       |
| `client/src/clientConfig.js`            | Anchor name, header color, support email, issuer account ID, Stripe publishable key and Connect ID. | Frontend (publicly accessible)   |
| `bridge.cfg`                            | Issuer account ID, database settings, horizon settings (testnet or live)              | Bridge                           |
| `client/public/.well-known/stellar.toml` | Issuer account ID                                                                     | Use `yourdomain` as issuer name  |

### Stripe keys

`serverConfig.js` has fields for your Stripe test keys. These are purely for developers to run integration tests and can be left blank. If you want to use Stripe test mode, then you should put your Stripe test keys in the `stripe` (not `stripeTest`) fields, and the same for `clientConfig.js`.

## Manual Testing

Stripe provides excellent tools for testing. If you run the application locally (`yarn dev` is simplest) using your stripe test keys, and have your default Stripe Connect redirect URI set to `http://localhost:3000`, then the application will work locally, hosted at `http://localhost:3000`.

To deposit, use any details and the first card specified [here](https://stripe.com/docs/testing#cards-responses), with any CVC and any expiry in the future. This will immeadiately add funds to your available balance.

To withdraw, click the connect button and sign up with a different email to your Stripe account. Use phone number `(000) 000-0000` and verification code `000-000`. You can use any US debit card as listed [here](https://stripe.com/docs/testing#cards). Then simply follow the instructions in the front-end.

## Running in Production

### Deployment

1. Build the front end:
```shell
cd client && yarn build && cd ../
```
2. Run the production server with `yarn server`. This will also run the bridge for you. If you want to host the bridge yourself, simply edit the bridge script in package.json to do nothing (for example `"bridge": " "`), and change the bridge payEndpoint in `serverConfig.js`.

HTTPS is required by Stripe.

### Finances
The first deposit will take 6 days to become available balance, so you will not be able to take withdrawals in this time. After that there will be a 2 day waiting time, although this can vary. I highly recommend you deposit some of your own funds via card or bank account and ensure this has entered available balance before you start accepting public deposits. A withdrawal freeze will not be popular even if it has good reason.

If a withdrawal fails due to insufficient balance, you will need to use the bridge reprocess payment feature to reinitiate the withdrawal once you have available balance. You should not do this within 24 hours of the failure or Stripes idempotency mechanisms will prevent the payment actually being resent, but you can always choose to pay the person manually from the Stripe dashboard and not reprocess the payment on the bridge.

### Oversight
The bridge GUI is at `http://localhost:8006/admin` and can be used to reprocess failed withdrawals as well as to view payment activity.

It is not possible to credit someones deposit without charging them, as the deposit code will authorize the charge, then send it, then capture it (which should never fail). It is also impossible to charge someone without crediting their deposit, as if this fails then the charge won't be captured.

For withdrawals, the failure mode to be aware of (excluding bad input from the person withdrawing, which you are not responsible for) is not having enough available balance to pay out. See [finances](#finances).

## Automated Integration Testing
I have written integration tests; they talk to Stripe, and they even use the Horizon testnet. They cover every path it is possible to cover without human input (it's impossible to go through a valid Connect flow programmatically), currently 93.4% of lines. To run them, ensure your bridge is set to use testnet, and that you have a valid (account existing) issuer ID and seed, and a test seed in `test/spec.js`. The accounts currently present in this repository do exist on testnet but may occasionally need topping up by the friendbot.

The tests use the `testStripe` keys from `serverConfig.js`, which should of course be your Stripe test keys, and if you want to be extra careful you should ensure that your production Stripe keys don't exist anywhere.

To run tests:

```shell
yarn test
```

Some tests might time out, in which case you can raise their timeouts in `test/spec.js`, and the tests which withdraw via the bridge may altogether fail as they are optimistically hoping that the bridge will notice the withdrawal and tell Stripe within a few seconds. This waiting time can be increased.