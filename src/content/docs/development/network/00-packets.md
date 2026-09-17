---
title: Packets
description: How to define packets
---

This tutorial will teach you how packets are defined in Steel.

## Implementing a Packet

For the purposes of implementing packets for Steel, there are two types of packets concerned:

- **Clientbound packets**: These packets are sent from the *server* (in our case, Steel) to the client.
- **Serverbound packets**: These packets are sent from the client to the server.

Packets are defined in `steel-protocol`.

### Clientbound Packet

Clientbound packets will need to implement the following two traits:
- `WriteTo` (we need to **write** data to send to the client); and
- `ClientPacket` (we need to be able to write the packet and know the ID corresponding to the packet)

In addition to the above, for deriving `ClientPacket`, we'll also need to specify the `#[packet_id(Play = ...)]` attribute.

For example, say we wanted to implement the `ClientboundEntityPositionSyncPacket` packet in *Minecraft*. The usual format for the file of a packet
is:

```
steel-protocol/src/packet/<category>/c_<packet>.rs
```
Usually, packets run in the "game" phase, so `category` is usually `game`.

Wwe can get `packet` by removing the `Clientbound` and `Packet` part of the *Minecraft*-named packet, and converting it to `snake_case`.
In our case, it would be `entity_position_sync`.

Here is how we can define our packet:

```rust
// /steel-protocol/src/packet/game/s_entity_position_sync.rs

// For packets that don't need custom writing logic, we can derive `WriteTo`.
#[derive(ClientPacket, WriteTo, Clone, Debug)]
// Specify the packet ID.
#[packet_id(Play = C_ENTITY_POSITION_SYNC)]
pub struct CEntityPositionSync {
    #[write(as = VarInt)]
    pub entity_id: i32,
    pub pos: DVec3,
    pub vel: DVec3,
    pub yaw: f32,
    pub pitch: f32,
    pub on_ground: bool,
}
```

:::caution
The derive will not work for types that don't implement `WriteTo`. You can either implement `WriteTo` for those types or manually implement
`WriteTo` for the entire packet.

It's also important to specify the properties of the packet in **the exact order** as that Minecraft uses (usually defined by a `StreamCodec`).
:::

Sometimes, the packet might be too complex or might be too unusual for the derive macro. In this case, you'll have to manually implement the `WriteTo` trait for the packet:

```rust
// /steel-protocol/src/packet/game/s_entity_position_sync.rs
pub struct CEntityPositionSync {
    // ...
}

// Manual implementation of WriteTo.
// This is actually exactly what the derive macro produces for the packet!
impl WriteTo for CEntityPositionSync {
    fn write(&self, writer: &mut impl Write) -> Result<()> {
        VarInt(self.entity_id as i32).write(writer)?;
        self.pos.write(writer)?;
        self.vel.write(writer)?;
        self.yaw.write(writer)?;
        self.pitch.write(writer)?;
        self.on_ground.write(writer)?;
        Ok(())
    }
}
```

:::note
The `#[write(...)]` attribute above is used to specify how a certain field is written. In this case, it tells Steel to write the property (here, `entity_id`)
as a `VarInt` (an integer encoded with a variable length). For more detail, check [Packet Traits](../01-packet-traits).
:::

### Serverbound Packet

Serverbound packets will need to implement the following two traits:
- `ReadFrom` (we need to **read** data from the data sent by the client); and
- `ServerPacket`

Unlike clientbound packets, we don't need an extra `packet_id` attribute to derive the required packet traits. Serverbound packet module names
start with an `s`, and their path looks something like so:
```
steel-protocol/src/packet/<category>/s_<packet>.rs
```

Here's an example for the `ServerboundUseItemPacket` packet:

```rust
// /steel-protocol/src/packet/game/s_use_item.rs
#[derive(ReadFrom, ServerPacket, Clone, Debug)]
pub struct SUseItem {
    pub hand: InteractionHand,
    #[read(as = VarInt)]
    pub sequence: i32,
    pub y_rot: f32,
    pub x_rot: f32,
}
```

The derive macro for `ReadFrom` is very similar to that for `WriteTo`, so the above also applies here.

:::note
Similarly to the `WriteTo` derive macro, we use the `#[read(...)]` attribute for more specific reading behavior of `ReadFrom`.
:::

## Scoping the Packet

Finally, we'll have to go to the appropriate module file of our packet and make it available to other modules.

```rust ins={3, 7}
// /steel-protocol/src/packet/game/mod.rs
// ...
mod c_entity_position_sync;
// ...

// ...
pub use c_entity_position_sync::CEntityPositionSync;
// ...
```

## Using a Packet

Steel can *send* clientbound packets to players and *receive* serverbound packets from players.

### Sending a Clientbound Packet

To send a clientbound packet to a player, we use the `send_packet` method of `Player`.

```rust {9, 10, 11, 12}
// /steel-core/src/command/builtins/tellraw.rs
fn send_message(context: &SteelCommandContext<CommandSource>) -> Result<i32, CommandSyntaxError> {
    // ...
    for target in targets {
        let message = message.try_resolve(&CommandTextResolver::with_entity_override(
            context.source(),
            target.as_ref(),
        ))?;
        target.send_packet(CSystemChat {
            content: message,
            overlay: false,
        });
    }
    // ...
}
```

### Receiving a Serverbound Packet

For play packets (which is most of them), you will need to add the handler for receiving the packet to the `decode_play_packet` function
in `JavaConnection`:

```rust
// /steel-core/src/player/connection/java.rs
fn decode_play_packet(packet: RawPacket) -> Result<DecodedPlayPacket, PacketError> {
    let data = &mut Cursor::new(packet.payload());
    let scheduled = |packet| DecodedPlayPacket::Scheduled(ScheduledPlayPacket(packet));

    Ok(match packet.id {
        // ...
        play::S_USE_ITEM => scheduled(
            ScheduledPlayPacketKind::UseItem(SUseItem::read_packet(data)?)
        ),
        // ...
    })
}
```

You will also have to add a new variant to the `ScheduledPlayPacketKind` enum and add the required match patterns.