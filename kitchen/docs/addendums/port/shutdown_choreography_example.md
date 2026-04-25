# Shutdown Choreography: A Coordinated Close Example

## 1. Introduction

This document illustrates a robust "shutdown choreography" for a `tofu`/`otofu`-based system. The goal is to demonstrate a graceful, negotiated shutdown that propagates cleanly from the application logic, through the **Conversation Layer**, and down to the **Transport Layer**.

This approach avoids abrupt connection termination, data loss, and unclean resource releases. It treats shutdown not as an error or a low-level I/O event, but as a formal, expected part of the protocol's lifecycle.

## 2. Layers & Actors

For this example, we have two actors, a **Client** and a **Server**. Each actor is composed of logical layers:

*   **Application Layer**: The user code that decides to initiate and orchestrate actions (e.g., `client.Shutdown()`).
*   **Conversation Layer**: Manages the state machine and rules of the dialogue (the `tofu`/`otofu` logic). It understands the *meaning* of messages.
*   **Transport Layer**: Manages the underlying I/O, sending and receiving raw bytes (e.g., a Proactor handling the socket). It has no knowledge of message content.

```
   Client                           Server
+-----------------+            +-----------------+
| Application     |            | Application     |
+-----------------+            +-----------------+
| Conversation    |<--Dialogue-->| Conversation    |
+-----------------+            +-----------------+
| Transport       |<---Bytes---> | Transport       |
+-----------------+            +-----------------+
```

## 3. The Message Protocol

To facilitate the negotiation, our message protocol includes specific shutdown-related message types.

```odin
// Odin-like pseudo-code
Message_Type :: enum {
    Normal_Data,
    Shutdown_Request, // Client -> Server: "I would like to close the connection."
    Shutdown_Ack,     // Server -> Client: "I have received your request and am ready to close."
}

Message :: struct {
    type: Message_Type,
    // ... other fields like payload, message_id, etc.
}
```

## 4. The Choreography: A Step-by-Step Guide

The following steps detail the sequence of events for a client-initiated shutdown.

### Step 1: Initiation (Client Application)

The process begins when the client's application logic decides it has completed its work.

*Client `main.odin` (pseudo-code):*
```odin
// ... client has finished its business ...
fmt.println("[Client App] Work is done. Initiating shutdown.")

// The application tells the Conversation Layer to begin the shutdown process.
// It does NOT directly touch the transport or socket.
client_conversation.shutdown()
```

### Step 2: The Request (Client Conversation Layer)

The `shutdown()` call is not a "close socket" command. It's a request to start the *conversation* about shutting down.

*Client `conversation.odin` (pseudo-code):*
```odin
shutdown :: proc(c: ^Conversation) {
    // 1. Change internal state. This prevents the application from sending more data.
    c.state = .Shutting_Down
    
    // 2. Create the special shutdown message.
    req_msg := Message{type = .Shutdown_Request}
    
    // 3. Send the message via the Transport Layer.
    fmt.println("[Client Conv] Sending Shutdown_Request.")
    transport.send(c.connection_id, req_msg)
    
    // 4. The Conversation Layer now waits for a Shutdown_Ack.
    // It will not and cannot close the connection yet.
}
```

### Step 3: Receipt and Interpretation (Server)

The server's Transport Layer receives the raw bytes, decodes them into a `Message`, and passes it up to the server's Conversation Layer for interpretation.

*Server `conversation.odin` (pseudo-code):*
```odin
// In the server's main message-handling loop...
handle_message :: proc(c: ^Conversation, msg: Message) {
    switch msg.type {
    case .Normal_Data:
        // ... process data as usual ...
    
    case .Shutdown_Request:
        fmt.println("[Server Conv] Received Shutdown_Request.")
        // The message is understood. Begin the server-side shutdown process.
        handle_shutdown_request(c)
        
    case .Shutdown_Ack:
        // A server should not receive an Ack. Log error.
    }
}
```

### Step 4: The Acknowledgement (Server Conversation Layer)

The server acknowledges the client's request. This is its opportunity to finish any in-flight work before agreeing to close.

*Server `conversation.odin` (pseudo-code):*
```odin
handle_shutdown_request :: proc(c: ^Conversation) {
    // 1. Change state to reject new work on this connection.
    c.state = .Shutting_Down
    
    // 2. (Optional but recommended) Ensure any final data is processed or flushed.
    // For example, wait for pending database writes for this client to complete.
    
    // 3. Create the acknowledgement message.
    ack_msg := Message{type = .Shutdown_Ack}
    
    // 4. Send the acknowledgement back to the client.
    fmt.println("[Server Conv] Sending Shutdown_Ack.")
    transport.send(c.connection_id, ack_msg)
    
    // 5. The server's side of the conversation is now complete.
    // It has fulfilled its part of the bargain and can safely request the
    // Transport Layer to close this specific connection.
    fmt.println("[Server Conv] Requesting transport to close connection.")
    transport.close(c.connection_id)
}
```

### Step 5: Completion and Finalization (Client)

The client's Transport Layer receives the `Shutdown_Ack`. The Conversation Layer can now conclude the process.

*Client `conversation.odin` (pseudo-code):*
```odin
// In the client's main message-handling loop...
handle_message :: proc(c: ^Conversation, msg: Message) {
    switch msg.type {
    // ...
    case .Shutdown_Ack:
        // The server has acknowledged our request to shut down. The negotiation is complete.
        if c.state == .Shutting_Down {
            fmt.println("[Client Conv] Received Shutdown_Ack. Closing connection.")
            
            // The Conversation Layer now instructs the Transport Layer to close the socket.
            transport.close(c.connection_id)
        }
    }
}
```
The `transport.close()` call in both the client and server will trigger the underlying socket closure and resource cleanup within the Transport Layer.

## 5. Key Architectural Benefits

This negotiated shutdown provides several crucial advantages:

1.  **No Data Loss**: The server gets a chance to process final messages before acknowledging the shutdown. The client waits for this acknowledgement, ensuring its request was received and handled.
2.  **Clean State Management**: Both parties transition through a `Shutting_Down` state, which provides a clean, predictable way to reject new work during the teardown process.
3.  **Resource Safety**: The Transport Layer is the sole owner of the socket resource. It only acts (closes the socket) when instructed to by the Conversation Layer, enforcing a clear chain of command.
4.  **Avoids TCP Race Conditions**: Abruptly closing one side of a connection can lead to network states like `TIME_WAIT` or `FIN_WAIT`, which can be problematic in high-traffic applications. A graceful close where both sides agree is fundamentally more stable.
5.  **Testability**: The shutdown logic is part of the formal protocol, meaning it can be tested with the same rigor as any other application feature.
