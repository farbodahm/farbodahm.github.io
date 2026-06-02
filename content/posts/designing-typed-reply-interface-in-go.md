---
author: "Farbod Ahmadian"
title: "Designing a Typed Reply Interface in Go"
date: "2026-06-02"
description: "Designing a typed reply API for a JSON-over-stdio protocol. Per-type methods, interfaces, and generics, with the trade-offs that come with each."
tags:
- golang
- distributed-systems
- developer-experience
---

The other night I did an old-school pair programming session with [Shaygan](https://glyphack.com/) on one of the [Build Distributed Systems](https://builddistributedsystem.com/) challenges. The problem itself was small: design a `msg.Reply()` API for a JSON-over-stdio protocol where every reply does roughly the same four things. The interesting part was how many different ways Go lets you factor that.

## The setup

Every message in the protocol looks like this:

```json
{
  "src": "c0",
  "dest": "n1",
  "body": {
    "type": "init",
    "msg_id": 1,
    "node_id": "n1",
    "node_ids": ["n1"]
  }
}
```

The outer shell is always the same: `src`, `dest`, `body`. Inside the body there's always a `type`, usually a `msg_id`, and sometimes an `in_reply_to`. The interesting fields vary per message: `init` has `node_id` and `node_ids`, `echo` has `echo`, broadcast has `message`, and so on.

Replying always does the same four things: swap `src` and `dest`, bump a counter for the new `msg_id`, copy the request's `msg_id` into `in_reply_to`, and write the result to stdout. I wanted that to be one line:

```go
msg.Reply(body)
```

## What I want from the types

Three things:

1. Each message stays typed. I want `msg.Body.NodeID`, not `msg.Body["node_id"].(string)`.
2. The `src`/`dest`/`msg_id`/`in_reply_to` fields are shared, not repeated per message type.
3. The reply API hides the boilerplate. No manual src/dest swap, no manual counter bump.

For (1) and (2), the shared shell is straightforward:

```go
type Envelope struct {
    Src  string `json:"src"`
    Dest string `json:"dest"`
}

type ReplyMsgBody struct {
    MsgID     int `json:"msg_id"`
    InReplyTo int `json:"in_reply_to,omitempty"`
}

type InitMessage struct {
    Envelope
    Body struct {
        Type    string   `json:"type"`
        MsgID   int      `json:"msg_id"`
        NodeID  string   `json:"node_id"`
        NodeIDs []string `json:"node_ids"`
    } `json:"body"`
}
```

`Envelope` is embedded into every message type. Its fields get promoted, so they appear flat in the JSON. New message types just embed Envelope and declare their own body. Pretty clean.

The hard part is (3). How do you write `Reply` once and have it work across `InitMessage`, `EchoMessage`, `BroadcastMessage`, and whatever else comes later?

We tried three approaches.

## Approach 1: One Reply method per message type

The first attempt: just write a Reply method on each typed message. No tricks.

```go
type InitOkBody struct{}

type InitOkMessage struct {
    Envelope
    Body struct {
        ReplyMsgBody
        InitOkBody
    } `json:"body"`
}

func (m *InitMessage) Reply(body InitOkBody, msgId int) {
    var msg InitOkMessage
    msg.Src = m.Envelope.Dest
    msg.Dest = m.Envelope.Src
    msg.Body.ReplyMsgBody = ReplyMsgBody{
        MsgID:     msgId,
        InReplyTo: m.Body.MsgID,
    }
    msg.Body.InitOkBody = body
    Log.PrintJSON(msg)
}
```

The call site is exactly what I wanted:

```go
msg.Reply(InitOkBody{}, nextMsgID())
```

Fully typed. If I pass an `EchoOkBody` by mistake, it won't compile.

The cost is duplication. For Echo, you write almost the same method again, just with different types in the slots. Six lines of swap-and-build, copy-pasted per message type. This is exactly what parameterized code (interfaces or generics) is for.

## Approach 2: Interfaces and a single Reply function

The classical Go way to write code that works across types is interfaces. Two small ones do it.

```go
type Message interface {
    Src() string
    Dest() string
    Type() string
}

type Body interface {
    Type() string
    Data() map[string]any
}
```

Every typed message implements `Message` with three one-line getters. Every reply body implements `Body` with its type string plus a `Data()` that returns the type-specific fields as a map.

```go
func (m InitMessage) Src() string  { return m.Envelope.Src }
func (m InitMessage) Dest() string { return m.Envelope.Dest }
func (m InitMessage) Type() string { return m.Body.Type }

func (b EchoOkBody) Type() string { return "echo_ok" }
func (b EchoOkBody) Data() map[string]any {
    return map[string]any{"echo": b.Echo}
}
```

A single `Reply` function works for everything:

```go
type Response struct {
    Envelope
    Body map[string]any `json:"body"`
}

func Reply(msg Message, body Body, msgId int) {
    response := Response{
        Envelope: Envelope{Src: msg.Dest(), Dest: msg.Src()},
        Body: map[string]any{
            "msg_id":      msgId,
            "in_reply_to": msgId,
            "type":        body.Type(),
        },
    }
    for k, v := range body.Data() {
        response.Body[k] = v
    }
    Log.PrintJSON(response)
}
```

Call site:

```go
Reply(msg, EchoOkBody{Echo: msg.Body.Echo}, nextMsgID())
```

One function, works for any pair. The boilerplate doesn't disappear, but it splits into single-line methods instead of repeated six-line blocks.

The trade-off is real though. The outgoing body becomes `map[string]any`, built by `Data()`. That means field names live as string literals (`"echo"`, `"node_id"`). A typo won't be caught until the grader rejects it.

## Approach 3: Generics

Generics let you keep approach 2's "one function, all types" win without the map-shaped escape hatch.

```go
type Replyable interface {
    Source() string
    Destination() string
    RequestMsgID() int
}

type ReplySetter interface {
    SetMsgID(int)
    SetInReplyTo(int)
}

type BodyCommon struct {
    Type      string `json:"type"`
    MsgID     int    `json:"msg_id"`
    InReplyTo int    `json:"in_reply_to,omitempty"`
}

func (b *BodyCommon) SetMsgID(id int)     { b.MsgID = id }
func (b *BodyCommon) SetInReplyTo(id int) { b.InReplyTo = id }

func Reply[B ReplySetter](req Replyable, body B, msgId int) {
    body.SetMsgID(msgId)
    body.SetInReplyTo(req.RequestMsgID())
    Log.PrintJSON(struct {
        Src  string `json:"src"`
        Dest string `json:"dest"`
        Body B      `json:"body"`
    }{req.Destination(), req.Source(), body})
}
```

Each typed body just embeds `BodyCommon`:

```go
type EchoOkBody struct {
    BodyCommon
    Echo string `json:"echo"`
}
```

Call site:

```go
Reply(msg, &EchoOkBody{
    BodyCommon: BodyCommon{Type: "echo_ok"},
    Echo:       msg.Body.Echo,
}, nextMsgID())
```

The type parameter `B` is the concrete body type. JSON gets marshaled from that concrete type, so `"echo"` is a struct tag, not a map key. The compiler tells you if the field is missing.

## Approach 4: Reflection (for Go < 1.18)

If you can't reach for generics, reflection gets you the same shape with one trade: the field check moves from compile time to runtime.

```go
func Reply(req Replyable, body interface{}, msgId int) {
    v := reflect.ValueOf(body).Elem()
    if f := v.FieldByName("MsgID"); f.IsValid() {
        f.SetInt(int64(msgId))
    }
    if f := v.FieldByName("InReplyTo"); f.IsValid() {
        f.SetInt(int64(req.RequestMsgID()))
    }
    Log.PrintJSON(struct {
        Src  string      `json:"src"`
        Dest string      `json:"dest"`
        Body interface{} `json:"body"`
    }{req.Destination(), req.Source(), body})
}
```

The body just needs `MsgID` and `InReplyTo` fields somewhere reachable. The cleanest way to guarantee that is to embed the same `BodyCommon` struct we used for generics:

```go
type EchoOkBody struct {
    BodyCommon
    Echo string `json:"echo"`
}

Reply(msg, &EchoOkBody{
    BodyCommon: BodyCommon{Type: "echo_ok"},
    Echo:       msg.Body.Echo,
}, nextMsgID())
```

`reflect.FieldByName` walks into embedded structs, so `MsgID` and `InReplyTo` are still found through `BodyCommon`. The call site is identical to approach 3, the body declarations are identical, and there's no `ReplySetter` interface to define or satisfy.

The cost: rename `BodyCommon.MsgID` to `BodyCommon.MessageID` and the code still compiles, still runs, just silently fails to populate the field. The risk is narrower than it sounds (it's one struct, not per-body), but the compiler can't help you catch it. You catch it at the grader.


## Wrapping up

This is the kind of problem where each approach is locally cleaner than the others depending on what you're optimizing for. Approach 1 keeps types end to end. Approach 2 keeps function count low. Approach 3 keeps both with generics. Approach 4 keeps both without generics, by moving the field check from compile time to runtime.

You can find the full code in [this PR](https://github.com/farbodahm/build-distributed-systems/pull/1/).
