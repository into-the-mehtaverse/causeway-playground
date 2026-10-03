# Causeway playground

Notes on [Causeway](https://www.netcore.network/causeway) from NetcoreNetwork: what it is, when a typical web app would use it, and what the discovery directory exposes. Checked against the public site and the `cwp2p` 0.2.2 client help text.

## What it is

Causeway is a local daemon, `cwp2p`, that connects two machines you trust and makes a TCP port on one of them appear as localhost on the other.

Each install creates its own ID from a local key. NetcoreNetwork is the directory those machines use to find each other. After the lookup, application traffic goes through an encrypted tunnel between the two devices. The default tunnel is WireGuard. Starting both daemons with `--data-channels` uses WebRTC instead. Both sides have to use the same mode. If a direct path is blocked, a relay can carry the encrypted packets.

The site describes an embeddable SDK. The public download is the client binary. There is no published library to import into an application. The network is on testnet, and Netcore says it will be open sourced at mainnet.

## When it fits a normal web stack

A Next.js frontend, a FastAPI backend, and Supabase do not need Causeway. Visitors reach FastAPI and Supabase because those services already have public addresses. A page hosted on Vercel cannot open localhost on a laptop at home.

Causeway fits when the program you want to reach is running on a specific computer that has no public address, and only machines you explicitly pair should reach it. A FastAPI app and database on a server at home are that case.

On the home server:

```bash
./cwp2p
./cwp2p port add 8000
```

On the laptop, after exchanging IDs:

```bash
./cwp2p peer add home <server-id> 8000:8000
```

`http://127.0.0.1:8000` on the laptop is FastAPI on the home server. The database stays on the server and talks to FastAPI over localhost. Map the database port only when a SQL client on the laptop should reach that same machine.

That works for you and for other machines you add with `peer add`. It does not make the API available to arbitrary browsers. A public site still needs a public path to the server, such as a port forward or a tunnel with a hostname.

The machines do not need to share Wi-Fi. Each one needs a path to the internet. Ethernet, Wi-Fi, or a hotspot all work. The same LAN works too.

## How discovery works

The ID is stored. It is stored as one signed note copied across the NetcoreNetwork nodes, which share a key-value store and agree on its contents with a consensus method Netcore calls RIPSTIK.

When the daemon starts, it publishes a signed note: this ID, this public key, and the address where the machine can be reached right now. The other daemon asks the network for that ID, checks the signature, and connects to the address in the note. A node can store and return the note. It cannot invent a new address, because it does not have the private key.

There is no Causeway account database. The ID is created on the machine. Today the directory is their testnet.

## Who can read a directory record

Anyone who knows the 64-character ID can read that record. Every testnet node that stores it can read it too. Lookup has no account and no allow-list, because the other machine has to fetch the note using only the ID.

The note contains:

- The public key.
- Current addresses, as ICE candidates. That includes the public IP and port, and sometimes a local or relay address.
- The ICE username and password for that signaling round.
- Enough testnet balance state for the network to accept the publication. Publishing costs a small testnet value, which is why `cwp2p charge` exists.

That is reachability data. Readers can see that the ID is online and where to send packets. They do not get FastAPI traffic, the database, files, or the private key. The private key stays in `cwp2p.json` on the machine. Tunnel traffic is encrypted and does not enter the key-value store.

Guessing IDs does not reveal the directory. The ID is 64 hex characters, a 256-bit value, so trying random IDs does not find a real one. A normal client lookup works only with the exact ID. Whether a client can list every ID is not documented. Node operators can read the notes their nodes already store, including addresses, without guessing.

The peer list from `peer add` is local configuration. The client help text does not show that list being written into the store.

## How an untrusted machine is kept out

Publishing the note does not open the machine. The daemon builds a tunnel only with IDs added locally:

```bash
./cwp2p peer add laptop <their-id>
```

The handshake requires the other side to prove it holds the private key for that ID. A machine that was never added fails the check, even if it looked up the address. In WireGuard mode, an added peer can reach a local service only after `port add` for that port. FastAPI stays bound to localhost. Nothing is forwarded in from the open internet.

`cwp2p.json` has to stay private. Copying it lets someone impersonate that ID.

## RIPSTIK

RIPSTIK is the agreement algorithm inside the directory. Applications do not call it.

Netcore describes it as leaderless and Byzantine fault tolerant. A Byzantine fault is a node that lies or answers different peers differently. Byzantine fault tolerance means the remaining nodes can still agree on one copy of the store while some nodes misbehave. They also say participation is tied to useful network work, described in an interview as a bandwidth resource barrier, rather than proof of work or proof of stake.

The public description stops there. The implementation is closed. No paper, proof, or code for RIPSTIK is public, and no patent under that name turned up. They call it novel. That claim cannot be checked from the outside. Leaderless Byzantine agreement and bandwidth-based participation both already exist as ideas elsewhere. RIPSTIK does not decide who may open a FastAPI port. The peer list and the key handshake do that.

## Private directory

The client can be pointed at a roster of nodes, with its own STUN and TURN servers. A roster that only your machines use would be a private directory: those IDs and addresses would sit on nodes you operate.

The public download is the client, and it joins Netcore's testnet. The node software is not released. Netcore says the network will be open sourced at mainnet. Nothing in the launch material offers a private instance or an enterprise on-prem product. Running a closed roster is a possible later shape. It is not something the current download can deploy.

Their stated uses are self-hosted apps, remote access to a home server or edge device, file sharing, cameras, IoT, and robotics. The directory publishes online status and IP address to ID holders and to the testnet nodes. A deployment that cannot leak that metadata would need control of the coordination nodes, which the current client does not provide.
