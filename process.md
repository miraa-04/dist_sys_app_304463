To understand how data moves, think of it as a Postal Service for Data. Your message doesn't just "teleport"; it is wrapped in multiple envelopes, each handled by a different "department" (layer) to ensure it reaches the right destination.

The Downward Journey (Sending)
When your Frontend sends a request to your Backend, the data travels "Down the Stack" through these four practical stages:

Application Layer (The Content): Your app creates the raw data (e.g., a JSON object or a gRPC call).

Transport Layer (The Tracker): Data is chopped into Segments. If using TCP, a header is added with a "Sequence Number" (to keep things in order) and a Port Number (so the OS knows which specific app gets the data).

Network Layer (The Address): The segment is put into a Packet. This adds the IP Address (the "Mailing Address" of the target computer).

Data Link/Physical Layer (The Delivery): The packet is put into a Frame with a MAC Address (the ID of the specific network card) and turned into raw Bits (electrical pulses or radio waves) to travel across the wire or air.

The Upward Journey (Receiving)
When the bits arrive at your Backend (running as a Windows service), the process happens in reverse, called Decapsulation:

Physical: The network card picks up the bits and reassembles the Frame.

Network: The OS looks at the IP Header. "Is this for me?" If yes, it strips the IP header and passes the Packet up.

Transport: The OS looks at the TCP Header. It checks the Port Number (e.g., 5000). It sees your Backend service is "listening" there.

Application: The final, original data is delivered to your Python/Node.js code.

The "Main Thing" to Remember
Logical Reality: Your Frontend "thinks" it is talking directly to the Backend code.

Physical Reality: The data must be wrapped (Encapsulated), sent across the physical hardware, and unwrapped (Decapsulated) at the other end.

In your setup with NSSM, if the Frontend can't talk to the Backend, the "break" is usually at the Transport Layer (wrong Port) or the Network Layer (wrong IP or a Firewall blocking the packet).
