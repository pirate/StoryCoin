## StoryCoin

StoryCoin is a collaborative storytelling experiment built around a pair of blockchains:

- an **outer blockchain** that stores the accepted sequence of words in the story, and
- an **inner vote chain** that tracks proof-of-work votes for each proposed next word.

Participants open the web UI, propose words, and use their browser to mine votes. The Rust node accepts the strongest vote chain for each candidate word and can advance the story by appending the most-voted word as the next block.

## Architecture overview

StoryCoin is split across three runtime pieces plus a small browser client:

1. **Rust blockchain node** (`/home/runner/work/StoryCoin/StoryCoin/src/server.rs`)
   - Maintains the authoritative `Blockchain` and in-memory vote state.
   - Accepts JSON commands such as `get_story`, `get_blocks`, `get_votes`, `set_blocks`, `set_votes`, and `choose_next_word`.
   - Stores one block per accepted story word.
   - Validates vote chains with a proof-of-work rule before accepting them.

2. **Python web server** (`/home/runner/work/StoryCoin/StoryCoin/webserver.py`)
   - Serves the Bottle application and the browser UI from `static/index.html`.
   - Accepts WebSocket connections at `/websocket/<name>`.
   - Relays messages between browsers and the Rust node through Redis queues.

3. **Python mesh-network wrapper** (`/home/runner/work/StoryCoin/StoryCoin/network.py`)
   - Starts the Rust node as a subprocess.
   - Connects to the `mesh-networking` library so nodes can exchange messages over UDP.
   - Uses `RedisProgram` to bridge network messages into Redis-backed queues.

4. **Browser client** (`/home/runner/work/StoryCoin/StoryCoin/static/index.html` and `/home/runner/work/StoryCoin/StoryCoin/static/js/`)
   - Displays the current story and proposed words.
   - Mines votes in the browser using JavaScript hashing.
   - Sends vote updates and story requests over a WebSocket.

### Core data model

- `Block` (`src/blockchain.rs`): stores a story word vote chain, a block id, and the previous block hash.
- `Blockchain` (`src/blockchain.rs`): the ordered list of story blocks. The longest valid chain wins.
- `Vote` (`src/wordvote.rs`): one proof-of-work vote for a candidate word.
- `VoteChain` (`src/wordvote.rs`): the chain of votes for a specific word. The longest valid vote chain wins for that word.

### Message flow

```text
Browser UI
  -> WebSocket
Python Bottle server
  -> Redis lists (node-recv / node-send)
Rust blockchain node
  -> optional UDP mesh propagation via network.py
Other peers
```

## Repository layout

- `/home/runner/work/StoryCoin/StoryCoin/src/lib.rs` - Rust library root
- `/home/runner/work/StoryCoin/StoryCoin/src/server.rs` - Rust node executable
- `/home/runner/work/StoryCoin/StoryCoin/src/blockchain.rs` - story blockchain logic
- `/home/runner/work/StoryCoin/StoryCoin/src/wordvote.rs` - vote-chain logic and proof-of-work checks
- `/home/runner/work/StoryCoin/StoryCoin/src/hash_utils.rs` - SHA-512 helpers
- `/home/runner/work/StoryCoin/StoryCoin/src/io_queue.rs` - Redis queue helpers
- `/home/runner/work/StoryCoin/StoryCoin/webserver.py` - Bottle + WebSocket server
- `/home/runner/work/StoryCoin/StoryCoin/network.py` - mesh-network entry point
- `/home/runner/work/StoryCoin/StoryCoin/static/` - HTML, JS, and CSS assets
- `/home/runner/work/StoryCoin/StoryCoin/tests/tests.rs` - Rust tests for hashing and blockchain behavior

## Prerequisites

You need the following installed locally:

- **Rust and Cargo** for the blockchain node
- **Python** plus the packages in `requirements.txt`
- **Redis** running locally on `127.0.0.1:6379`

The current code assumes Redis is reachable at `redis://127.0.0.1/` and that the web server can bind to port `80`.

## Build instructions

### Build the Rust node

```bash
cargo build
```

This produces the blockchain server binary at:

```text
target/debug/server
```

### Install Python dependencies

```bash
pip install -r requirements.txt
```

## Usage

### 1. Start Redis

Start a local Redis server using your platform's usual command, for example:

```bash
redis-server
```

### 2. Build the Rust node

```bash
cargo build
```

### 3. Start the web server

```bash
python webserver.py
```

The server listens on `0.0.0.0:80` and serves:

- `/` - a short help message
- `/<name>` - the StoryCoin UI for a participant name
- `/websocket/<name>` - the WebSocket endpoint used by the UI

Example:

```text
http://localhost/alice
```

### 4. Interact with the story

From the browser UI you can:

- view the current story,
- propose a new word,
- mine votes for a word in the browser,
- see the list of connected participants, and
- manually advance the story to the most-voted next word.

### Optional: start the mesh-network node

If you want to run the mesh-network wrapper instead of just the web server:

```bash
python network.py
```

Notes:

- `network.py` starts `target/debug/server`, so the Rust binary must already exist.
- The script currently creates `UDPLink('en0', 2010)`, so you may need to adjust the interface name for your machine.

## Testing

Run the existing Rust tests with:

```bash
cargo test
```

At the time this README was updated, the existing test suite passed locally.

## How dependencies are used

### Rust dependencies (`Cargo.toml`)

- `serde`, `serde_derive`, `serde_json`
  - Used to serialize and deserialize blocks, vote chains, and messages exchanged with the Python layer.
- `rust-crypto`
  - Provides the SHA-512 implementation used by `hash_utils.rs` and vote/block hashing.
- `redis`
  - Used by `src/io_queue.rs` so the Rust node can read commands and publish responses through Redis lists.
- `rand`
  - Included as a Rust dependency, though the current checked-in code does not appear to use it directly.
- `nix`
  - Included for Unix-oriented functionality; imported in `src/server.rs` but not actively used in the current code.
- `time`
  - Included for time-based behavior; there are commented-out timing hooks in `src/server.rs`.

### Python dependencies (`requirements.txt`)

- `redis`
  - Python Redis client used by `webserver.py` to pass WebSocket messages to the Rust node.
- `gevent` and `gevent-websocket`
  - Provide the WSGI server and WebSocket support used by the Bottle app.
- `bottle`
  - Serves the HTML UI and static assets.
- `mesh-networking`
  - Supplies `Node`, `UDPLink`, and `RedisProgram` for peer-to-peer networking in `network.py`.

### Frontend libraries

The browser UI loads a few third-party libraries directly from CDNs in `static/index.html`:

- jQuery slim
- Tether
- Bootstrap JavaScript

It also serves local static assets such as `bootstrap.min.css`, `mining.js`, and the animated background scripts from `/static/`.

## Development notes and caveats

- The Rust node and Python web server communicate through Redis list keys named `node-recv` and `node-send`.
- The browser mines vote nonces client-side, so multiple open browsers can contribute proof-of-work votes.
- The project is an experiment and still contains a few hard-coded assumptions, including Redis on localhost, port `80`, and the `en0` network interface in `network.py`.
