# Bussn

Bussn is a portable message broker for sharing durable messages across application instances and JavaScript runtimes.

## Language

**Broker Domain**:
A named logical message space coordinated through one authoritative Store. Independent Store authorities define independent Broker Domains.

**Broker**:
The application-facing Effect module connected to one Broker Domain. It is neither a running server nor a durable authority.

**Message**:
An immutable, uniquely identified record created by a successful publication to one Destination. Redelivery and replay refer to the same Message; intentional republication creates another Message.

**Destination**:
An opaque address within one Broker Domain. Each Message has exactly one Destination; publishing to several Destinations creates separate Messages without implied atomicity.

**Subscription**:
A named, durable recipient within one Broker Domain, bound to exactly one Destination. It owns delivery state independently of other Subscriptions and continues to exist without active Consumers.

**Consumer**:
A temporary, scoped participant attached to one Subscription. It may claim Deliveries but owns no durable progress; ending it neither deletes its Subscription nor acknowledges its work.

**Delivery**:
One Subscription's uniquely identified, independent handling of one Message. It owns the mutable claim and completion state without copying or changing the Message.

**Claim**:
The Store-recorded, time-limited right for one Consumer to process one Delivery. A Delivery has at most one active Claim; expiry permits another Consumer to claim it without guaranteeing that the previous Consumer stopped processing.

**Acknowledgement**:
The Store-recorded, final completion of one Delivery by the Consumer holding its active Claim. It prevents redelivery to that Subscription without affecting the Message or its Deliveries to other Subscriptions.

**Store**:
The sole authority for a Broker Domain's Messages, Subscriptions, and delivery state. It is not an authority for application, workflow, event-source, or audit state.

**Relay**:
The live transport used to deliver Messages and exchange delivery-related communication between Broker participants. It owns no durable state; interrupted or uncertain communication is recovered through the Store.
