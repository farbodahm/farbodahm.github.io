---
author: "Farbod Ahmadian"
title: "Designing a Typed Reply Interface in Go (Part Two): Dispatching Incoming Messages"
date: "2026-06-09"
description: "The other half of the typed message problem: reading one line of JSON and turning it into the right concrete Go type. Sealed interfaces, peek-then-decode, and a scanner-style API."
tags:
- golang
- distributed-systems
- api-design
---

[The first part](/posts/designing-typed-reply-interface-in-go) was about replying: given one known message type, build the matching response. That direction is 1:1 and deterministic, `InitMessage` to `InitOk`, `EchoMessage` to `EchoOk`.

This part is the other direction, and it's where the types get interesting. A single line arrives on stdin, it could be any of N message types, and I have to turn it into the right concrete Go value before I can do anything with it. That's the dispatch problem I flagged at the end of part one. Here's how I solved it.

## The sealed set

First I need a type that means "one of the messages I know how to read." Go doesn't have sum types, but the sealed interface pattern gets close: an interface with an unexported method, which only types in my own package can implement.

```go
type Incoming interface {
    isIncoming()
}

func (InitMessage) isIncoming() {}
func (EchoMessage) isIncoming() {}
```

The method does nothing. It exists only to tag a type as a member of the set. Because it's unexported, nobody outside the package can add a member, so a type switch over an `Incoming` has a known, closed list of cases.

## Peek, then decode

JSON can't unmarshal into an interface. It has no way to know which concrete type to build. But the wire format tells me, in the body's `type` field. So I read the line once to look at that field, then unmarshal a second time into the type it names.

```go
func decodeByType(t MessageType, line []byte) (Incoming, error) {
    switch t {
    case MsgTypeInit:
        var m InitMessage
        err := json.Unmarshal(line, &m)
        return m, err
    case MsgTypeEcho:
        var m EchoMessage
        err := json.Unmarshal(line, &m)
        return m, err
    default:
        return nil, fmt.Errorf("unknown message type: %q", t)
    }
}
```

Two passes over the same bytes. The first reads one field and is cheap; the second does the real work. This switch is the one place that grows by a few lines per message type. That's the inherent cost of dispatch, and for now I'd rather have it sitting here in plain sight than hidden inside a registry. More on that at the end.

## Handing the value back

Now the API question. `decodeByType` returns an `Incoming`, but how should the caller loop over a stream of them? I tried three shapes.

The first returns a tuple:

```go
for {
    msg, ok := ScanTyped()
    if !ok {
        break
    }
    // switch ...
}
```

It works, but the break-on-EOF dance is noise I'd repeat in every challenge.

The second is a channel I can range over:

```go
for msg := range Messages() {
    // switch ...
}
```

Nicer to read, but it puts a goroutine behind stdin. If the consumer ever stops early, that goroutine blocks forever on the next read. For a loop that always runs to EOF it's harmless, but it's machinery I don't need.

The third is the one I kept. Pass a pointer and mirror `bufio.Scanner`:

```go
var msg Incoming
for ScanTyped(&msg) {
    // switch ...
}
```

The trick is that `ScanTyped` already knows the concrete type from the peek, so it assigns that value straight into the interface:

```go
func ScanTyped(target *Incoming) bool {
    for stdin.Scan() {
        line := stdin.Bytes()
        if len(line) == 0 {
            continue
        }
        var probe struct {
            Body struct {
                Type MessageType `json:"type"`
            } `json:"body"`
        }
        if err := json.Unmarshal(line, &probe); err != nil {
            Log.Error("parse JSON: %v", err)
            continue
        }
        msg, err := decodeByType(probe.Body.Type, line)
        if err != nil {
            Log.Error("%v", err)
            continue
        }
        *target = msg
        return true
    }
    return false
}
```

Assigning a concrete value to a `*Incoming` is plain Go. No reflection, no goroutine. Note that this only works because of the peek: unmarshaling directly into `*Incoming` would still fail, since `json` wouldn't know the type. Looking at `type` first is what makes the assignment possible.

## Putting it together

Here's the whole echo service. Init records the node's identity and replies `init_ok`; echo sends the payload back. `Reply` from part one handles the src/dest swap and the `msg_id` bookkeeping, so each case is one line of intent.

```go
func main() {
    node := NewNode()

    var msg Incoming
    for ScanTyped(&msg) {
        switch m := msg.(type) {
        case InitMessage:
            node.Init(m.Body.NodeID, m.Body.NodeIDs)
            Log.Info("initialized node %s with peers %v", node.ID, node.Peers)
            node.Reply(m, &InitOkBody{})
        case EchoMessage:
            node.Reply(m, &EchoOkBody{Echo: m.Body.Echo})
        }
    }
}
```

The reading side and the writing side meet in that switch. Adding a new message type costs three small edits: one case in `decodeByType`, one marker method, one branch here. None of them touch the loop or the `Reply` code.

## Next step: when the switch starts to hurt

Two cases is fine. But I can already see where this goes. By the time I'm handling broadcast, gossip, reads, and all their acks, both switches are a dozen cases long, and every new message type means editing the same two places in lockstep. That repetition is the signal that the explicit switch has done its job and it's time for something else.

The something else is a registry: a map from type string to handler, instead of a switch.

```go
handlers := map[MessageType]Handler{
    MsgTypeInit: handleInit,
    MsgTypeEcho: handleEcho,
}
```

Routing becomes a map lookup, handlers can live in their own files, and adding a message type is one registration line rather than edits spread across two switches. Maelstrom's own Go library is built this way: you call `node.Handle("echo", fn)` and it dispatches by the type string at runtime.

I'm not reaching for it yet. With a few message types, a switch reads better than a map of functions, and I'd rather not add the indirection before the pain is real. But I know the move, and I'll make it the day the switch gets long enough to annoy me.
