# librats API Reference {#mainpage}

Welcome to the **librats** API documentation. librats is a high-performance,
lightweight peer-to-peer networking library written in C++17, with a stable C
ABI and language bindings.

## Quick Links

- [Main Website](https://librats.com)
- [GitHub Repository](https://github.com/DEgITx/librats)
- [Architecture overview](https://github.com/DEgITx/librats/blob/master/ARCHITECTURE.md)

## The model in one minute

The entry point is librats::Node — a thin, secure transport **core**. On its own
a Node gives you an encrypted transport (Noise_XX) over TCP or UDP, a
self-certifying librats::PeerId, manual dialing, a peer directory, and raw
channel messaging. Everything else — discovery, pub/sub, file transfer, NAT
traversal, reconnection — is an opt-in librats::Subsystem you attach with
`add_subsystem()` **before** `start()`. You pay only for what you attach.

```cpp
#include <librats/node/node.h>
#include <librats/subsystems/dht_discovery.h>
#include <librats/subsystems/message_json.h>

using namespace librats;

int main() {
    NodeConfig config;
    config.listen_port = 8080;

    Node node(config);
    node.add_subsystem(std::make_unique<DhtDiscovery>(DhtDiscovery::Config{}));
    node.add_subsystem(std::make_unique<MessageJson>());

    // raw bytes on a named channel
    node.on("chat", [](const Peer& from, ByteView data) {
        std::printf("%s: %.*s\n", from.id().short_hex().c_str(),
                    (int)data.size(), (const char*)data.data());
    });

    // typed JSON messages (via the MessageJson subsystem)
    node.json()->on("hello", [](const PeerId& from, const librats::Json& j) {
        std::printf("hello from %s: %s\n", from.short_hex().c_str(),
                    j.value("text", "").c_str());
    });

    if (!node.start()) return 1;
    node.connect("192.168.1.100", 8080);
    // ... run ...
    node.stop();
}
```

## Core types

- librats::Node — the P2P node facade (transport core + subsystem host)
- librats::NodeConfig — node construction options
- librats::Peer / librats::PeerId / librats::PeerInfo — peer identity and handles
- librats::Subsystem / librats::PeerNetwork — the contract every subsystem builds on
- librats::EventBus / librats::ServiceRegistry — node-scoped coordination buses
- librats::MessageType — inner-message kinds on the wire

## Messaging surfaces

| Surface | API | Payload |
|---------|-----|---------|
| Raw channel | `node.on/send/broadcast(channel, …)` | bytes |
| Typed JSON | `node.json()->on/send(type, …)` (librats::MessageJson) | librats::Json |
| Pub/sub topics | librats::PubSub `subscribe/publish` | bytes, GossipSub mesh |

## Subsystems

- librats::DhtDiscovery — Kademlia/Mainline-DHT peer discovery
- librats::MdnsDiscovery — local-network discovery
- librats::PubSub — GossipSub topic pub/sub
- librats::MessageJson — typed JSON messaging
- librats::FileTransfer — streaming file/directory transfer with integrity
- librats::PingService — liveness / RTT probing
- librats::PortMappingService — automatic UPnP IGD + NAT-PMP port forwarding
- librats::HolePunch — UDP hole punching through a shared peer (provides librats::HolePunchService)
- librats::Relay — last-resort connectivity: the connection is carried through a peer both ends reach, still encrypted end to end (provides librats::RelayService)
- librats::PeerExchange — PEX: grow the mesh from peers' peers
- librats::ReconnectionService — re-dial dropped peers with backoff
- librats::StorageManager — distributed key-value store (requires `RATS_STORAGE`)
- librats::Bittorrent — BitTorrent client sharing the node's DHT (requires `RATS_SEARCH_FEATURES`)

## Security

Connections use the Noise_XX_25519_ChaChaPoly_SHA256 handshake by default
(mutual authentication, perfect forward secrecy). The application protocol
name/version is bound into the handshake prologue, so apps that disagree cannot
connect. Set `NodeConfig::security` to `Plaintext` to disable encryption.

## C ABI

For FFI and language bindings, the canonical C API is declared in
[bindings/rats.h](@ref rats.h): an opaque `rats_t` wraps a Node, subsystems are
enabled with `rats_enable_*()` before `rats_start()`, and fallible calls return
`rats_error_t`.

## Build Configuration

| Option | Description |
|--------|-------------|
| `RATS_BUILD_TESTS` | Build unit tests (GoogleTest) — default ON |
| `RATS_BUILD_CLIENT` | Build the `rats-client` reference application — default ON |
| `RATS_BUILD_EXAMPLES` | Build the `examples/` programs — default OFF |
| `RATS_BINDINGS` | Build the C ABI — default ON |
| `RATS_INSTALL` | Generate install/export targets (`find_package(rats)`) — default ON |
| `RATS_STORAGE` | Enable distributed key-value storage — default OFF |
| `RATS_SEARCH_FEATURES` | Enable BitTorrent features — default OFF |
| `RATS_SHARED_LIBRARY` / `RATS_STATIC_LIBRARY` | Library linkage — default static |
| `RATS_ENABLE_ASAN` / `RATS_ENABLE_TSAN` | Sanitizer builds (mutually exclusive) — default OFF |

## License

librats is released under the MIT License.
