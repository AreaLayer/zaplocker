![](https://supertestnet.github.io/swap-service/zaplocker-logo.png)

Non-custodial lightning address server with base layer support too

# How to try it

Go here: https://zaplocker.com

# Video

[![](https://supertestnet.github.io/swap-service/zaplocker-with-youtube-logo.png)](https://www.youtube.com/watch?v=5LYt4H6xA-g)

# Four problems zaplocker solves

(1) End users don't want to run a web server. But lightning addresses don't work without a web server where wallets can request lightning invoices. Zaplocker allows a service provider to run a web server on their users' behalf.

(2) Lightning address servers usually take custody of user funds. Due to problem #1, most users rely on the following public lightning address servers: getalby.com, stacker.news, and walletofsatoshi.com. But the default mode of these servers is custodial. Custodying user funds introduces many costs and risks for lightning address servers, including: risk of theft, risk of accidental loss, cost of complying with financial custody laws, and risk of arrest if you do not comply with those laws. Zaplocker lets anyone provide lightning addresses to their users without taking custody of user funds, thus reducing those risks.

(3) Avoiding third-party custody usually means running an always-on lightning node. Due to problem #2, some lightning address servers have a feature that lets users receive funds directly to their own lightning node. This trades off the custody problem for an always-on problem: if your user only has a lightning wallet on their phone, it has to be "on" (i.e. open) or it can't give the lightning address server a lightning invoice when someone requests one. Meaning many payments just fail before they even start. Zaplocker lets users avoid third-party custody *without* an always-on lightning node.

(4) Lightning addresses don't usually work on the base layer. Many people want their payments to go straight to a cold storage wallet or an exchange account or a multisig vault. Lightning addresses are, as the name implies, lightning tech, which means they send funds to a "hot wallet" exposed to the internet -- because *all* lightning wallets are exposed to the internet. Zaplocker gives users a choice: do you want to receive your payments into a lightning wallet or a bitcoin address on the base layer? Which means lightning addresses, for the first time, support the base layer.

# How it works

Zaplocker takes my old [hodl contracts](https://github.com/supertestnet/hodlcontracts) idea and applies it to lightning addresses. A user creates an account with zaplocker by logging in with nostr and choosing a username. When they log in, they -- in the background -- create a bunch of lightning payment hashes and share them with the server. The server then shows the user their lightning address and a "pending payments" dashboard.

If someone requests an invoice from that user, zaplocker creates a "hodl invoice" using one of the user's payment hashes. This means zaplocker cannot settle any payment sent using that invoice. Zaplocker simply does not have the keys, the user alone has them. If the server detects that the sender *tried* to pay that hodl invoice, it does not fail the payment right away. Instead, it uses nostr to notify the user that they have a pending payment and invites them to visit zaplocker to settle it within the next 16 hours. When the user arrives at zaplocker, they can see their pending payment and two options: settle on lightning or settle on the base layer.

To settle on lightning, the user must create a "custom lightning invoice" whose payment hash matches the one held by the server. Few lightning wallets let you do this, but LND does. Users can use that for now, and I am currently in talks with other lightning wallet developers to add support for custom lightning invoices. If the user supplies a compatible lightning invoice for the same amount as the one held by the server (minus a service fee), zaplocker will pay it. This will give the server a "proof of payment" which consists of the very key they need in order to settle the sender's payment. Settling that payment completes the circuit -- the end user got paid, zaplocker got its service fee, and zaplocker never had custody of the user's funds.

To settle on the base layer, the user must say what bitcoin address they want their money to end up in. The user will then coordinate a submarine swap with the server. The server deposits the right amount of money (minus a service fee) to a "swap address" on bitcoin's base layer, and the user sweeps the money out of that address into the bitcoin address they picked. But the user can only sweep the money by revealing to the server the key the server needs in order to settle the sender's payment. (If the user neglects to reveal that key, the server gets their deposit back after 10 bitcoin blocks.) Assuming the user sweeps their money, that completes the circuit -- the end user got paid, zaplocker got its service fee, and zaplocker never had custody of the user's funds.

# Other cool things about zaplocker

- Zaplocker is free and open source software. You can run it yourself to provide users of your service with a lightning address without taking custody of user funds. The zaplocker name as well as the software is fully released into the public domain, with no rights reserved. If you're a developer, go hog wild! Make it your own! Do it your way, with your own spin, on your own server!
- Zaplocker supports zaps. If you add your zaplocker address to your nostr account, people can zap you on nostr-based social networks. Zaplocker sends out a zap receipt when a lightning payment goes into a pending state, and zappy apps can detect this receipt to show a green checkmark to their users -- without waiting for the recipient to come online and settle the payment.

# Fixing the man in the middle

There is an attack that lightning address servers such as zaplocker can do to steal funds from a sender. Show the sender a lightning invoice where the *server* holds the keys instead of the recipient, then settle the sender's payment and never tell the intended recipient about it. Zaplocker proposes solving this problem by signing and broadcasting a bunch of nostr messages at various stages of a payment. This solution is implemented in zaplocker. 

When a user logs in for the first time, they create a bunch of payment hashes for the server to use when generating lightning invoices. Zaplocker has the user *sign their payment hashes* using their nostr public key and displays the user's signature and nostr public key on the endpoint where invoices are generated. Sending wallets can *validate that signature* before sending the payment, that way the sender knows the user has the keys.

However, this solution is only partial: wallets need to implement it, and even if they do implement it, the server can still steal funds by reusing a "used" payment hash once they know its preimage. This is because once a payment hash has been "used" the server knows the key now. If they "reuse it," the signature on that payment hash will still be valid, but now the server can use the key -- which they now know -- to settle the payment and never tell the intended recipient about it. To fix that, zaplocker proposes doing this: when a sender sends a payment, they should send a nostr note announcing that that payment hash has been "used up."

This note should be sent publicly to a set of nostr relays selected and signed by the recipient, and this note should reference a public key created by treating the payment hash as a private key and deriving the corresponding public key. Senders can request this message from the user's nostr relays before sending a payment, and refuse to send the payment if they detect a message from a previous sender stating that the payment hash was used. This solution will require zaplocker to *expand* the amount of info signed by the user when they log in. Namely, they will need to sign a message stating what relays to send these notes to, and zaplocker will need to display that message as well as the user's signature so that senders can validate it and know what relays to listen for messages on. Also, sender wallets will need to implement the message-sending function and validate that at least one of those relays is still working.

However, even *this* solution is only partial. Aside from the fact that wallets still need to implement it (it's already implemented in zaplocker but that's only half the battle), the server can *still* steal funds by forwarding a smaller amount of funds to the user than the amount intended by the sender, without telling the recipient about the difference. To fix that, zaplocker proposes that the recipient's LUD06 callback endpoint should specify what pubkey they expect zaplocker to use in its invoices and display a signature by the recipient that is valid over a hash of that pubkey, senders should validate the recipient's "keyhash" signature and ensure the invoice they are given is signed by the zaplocker's pubkey, and the sender's note should *also* contain the invoice generated by the server. The recipient's browser should then *listen for that note* before settling a payment, and ensure (1) such a note exists (2) its pubkey belongs to zaplocker and (3) the amount in the invoice is equal to the amount being forwarded by the server (minus the agreed-upon service fees). Zaplocker implements all of these solutions, but until wallets implement them, users must trust the zaplocker server not to steal money in any of the above mentioned ways -- and even after they are implemented in *some* wallets, users still have to trust zaplocker not to steal from senders who *don't* implement this proposal.

# Protocol

1. Suppose Alice wants a lightning address, Bob wants to provide a zaplocker-like service, and Carol wants to pay Alice. The protocol begins when Alice gives Bob (1) a nostr pubkey that is publicly associated with Alice, (2) 1000 randomly generated payment hashes, (3) 1000 signatures that are valid for Alice's pubkey where the message hash, for each signature, is one of the payment hashes, (4) a set of nostr relays that Alice wants to use in this protocol, and (5) a signature that is valid for Alice's pubkey where the message hash is created by concatenating all of the nostr relays Alice sent to Bob and hashing the result. At this point Alice also ought to choose a lightning address username, let us suppose she chooses the username "alice." Bob ought to store each payment hash, signature, and Alice's set of relays in an account that associates Alice's pubkey with Alice's username. Then Bob should create an endpoint at bobsdomain.com/.well-known/lnurlp/alice that serves a json object structured acording to [the LUD06 standard](https://github.com/lnurl/luds/blob/luds/06.md). Yay, Alice now has the lightning address "alice@bobsdomain.com" -- she may add it to her nostr profile if she wants or put it on her business card or do whatever she wants with it.

2. If Carol wants to pay Alice, Carol should visit Alice's LUD06 endpoint. She should parse the LUD06 json object and query its callback url with an "amount" parameter indicating how many millisats she wants to pay Alice. Bob should then create a lightning invoice that can only be settled using one of Alice's payment hashes, ensure its amount is identical to the amount Carol requested, and present it to Carol in a json object at the callback endpoint Carol requested it from. That json object should also contain an array consisting of Alice's relays, a signature by Alice that is valid over a message hash which is the lightning invoice's payment hash, a signature by Alice that is valid over a message hash which is created by concatenating all of the nostr relays Alice sent to Bob and hashing the result, a hash of Bob's pubkey, and a signature by Alice that is valid over a hash of Bob's pubkey.

3. Carol should abort if (1) the relay signature is not valid for Alice's set of relays, (2) the payment signature is not valid for the payment hash in the lightning invoice, (3) the amount in the invoice is not the amount Carol requested, (4) Alice's "keyhash" signature is not valid for the keyhash presented by Bob, or (5) the pubkey of the invoice Bob gave to Carol does not hash to the keyhash presented by Bob. If Carol does not abort, she should validate that at least one of Alice's relays is still operative. I recommend doing this by (1) sending a test message to each relay and (2) trying to retrieve that test message from each relay. If no relay can return the test message, Carol should abort. If Carol does not abort, her wallet should present Alice's pubkey to her and ask her to confirm that's who she really wanted to pay. It may be helpful to look up Alice's profile on her relays, but keep in mind that a picture and display name is not enough to determine who Carol is paying because those are easily spoofed. Carol should be shown Alice's pubkey and asked to validate that she checked a trustworthy source to ensure that pubkey belongs to Alice. If Carol confirms, she should also derive Alice's payment hash from Bob's invoice, treat it like it is a nostr private key, derive its corresponding public key, and make a request to all of Alice's nostr relays, asking them for any kind 55869 messages referencing the public key derived from Alice's payment hash in a "p" tag. She should abort if any of Alice's nostr relays returns a response containing a nostr event whose content field consists of a valid lightning invoice whose pubkey is the same as the pubkey of the invoice Bob gave to Carol in paragraph 2, because that means Bob is trying to reuse an already-used payment hash. That is bad because if the payment hash is used, Bob may already know its preimage, which would allow him to settle Carol's payment to Alice without forwarding the funds to Alice. If Carol does not abort, she should generate an ephemeral nostr keypair (a new one for each payment) and use it to send a kind 55869 nostr note to Alice's relays. Its content field should contain only one thing: the lightning invoice signed by Bob. Its tags should only contain a "p" tag referencing the public key generated from Alice's payment hash. After submitting this event, Carol should attempt to pay the invoice Bob gave her.

4. Note that if Bob gave Carol a lightning invoice that failed any of the checks in the first sentence of paragraph 3, Carol has no way to prove that Bob did anything fraudulent. She cannot prove that Bob provided her with an invalid signature, a lightning invoice with an "unsigned" payment hash, or a lightning invoice for the wrong amount. If she tries to warn others that Bob tried to scam her, her warnings should be ignored because she has no proof. Moreover, it is not true: Bob did not try to scam her. Bob simply followed a different protocol that Carol doesn't know, and therefore he offered something to Carol that she did not want to pay. "Doing something other than what you want" is not a scam. "Following a different protocol" is not a scam. Bob made no promise that he would follow the protocol outlined in this document, and since he is not breaking a promise, he is not scamming. If I walk into a store and ask for a bottle of milk and the merchant offers me a dozen eggs instead, that is not a scam, it is a counteroffer. If I don't like the counteroffer, I should just not pay for it. It would be illogical for me to warn other people that the merchant is a scammer just because he didn't do what I wanted unless he made a promise to do what I wanted and then broke that promise. It would be similarly illogical for Carol to warn others that Bob is a scammer just because he isn't following the protocol outlined in this document. For all Carol knows, Alice is using this service provider for a protocol that Carol simply doesn't know about.

5. Bob's lightning node should detect the payment Carol sent in paragraph 3 without the ability to settle it. (He does not have the necessary preimage.) However, he should hold the payment in an "accepted" state and notify Alice that she has a pending payment. The notification method is outside the scope of this document, but in my initial implemention I chose to notify Alice via a nostr dm. Other options I considered are via email and via social media. Upon receiving this notification, Alice should visit Bob's website and log into her account. She should see her pending payment and an offer by Bob to settle it by providing a lightning invoice with the same payment hash as the pending payment. Before doing so, Alice should make a request to her relays asking for any kind 55869 events referencing the pubkey derived from her payment hash. She should check each such event to see if its content field contains a valid lightning invoice whose pubkey matches Bob's. If she discovers two different invoices created by Bob with identical payment hashes, Alice should not only abort -- her client should also notify her that Bob tried to scam her and provide her with both of the invoices Bob created so that she can prove to the world that Bob is a scammer.

6. Paragraph 5 deals with what to do if there are at least two responses from Alice's relays. This paragraph will deal with what to do if there are zero or one. If there are zero such responses, Alice should interpret that to mean the sender did not follow the protocol outlined in this document. In that case, Alice should send the event to her relays that the sender was *supposed* to send -- the kind 55869 event discussed in paragraph 3, except now without the invoice, just a blank content field instead -- and she should realize that Bob may send her *less* than the amount the sender wanted to give her. That is because the sender did not *announce* the amount he intends for her by sending the kind 55869 message, so Alice has no way to validate that the amount being forwarded to her is correct. She should abort if she does not trust the relay. However, even if there is one response, that only means one sender *did* follow this protocol. Alice should realize that there may be a *second* sender who did not follow this protocol, an "undetected sender" to whom Bob may have given an invoice with the same payment hash as the one he gave to the sender she knows about. Therefore, in the case of zero or one such responses, if Alice chooses to settle her payment, she necessarily puts at least some trust in Bob: there may have been one or more senders who did not follow this protocol and to whom Bob gave invoices with the same payment hash as the one he gave to the sender she knows about, and those senders may have pending payments that Bob is not telling Alice about. If such senders exist and Alice settles the payment she *does* know about, Bob can steal from the undetected senders. She must trust him not to do that, or, if she does not trust him, she may abort.

7. If Alice is okay with the level of trust outlined in paragraph 6, she should make an invoice for the same amount as the one Bob told her (minus any service fees Bob may choose to keep for himself) and use the same preimage. She should disclose this invoice to Bob, who should validate that (1) the amount is what he expects and (2) the CLTV locktime on Alice's invoice is less than the remaining time on the payment from the sender to himself. If either of those checks fails, he should abort. If he does not abort, he should pay Alice's invoice. If Alice settles the invoice, Bob will necessarily get Alice's preimage, which he can now use to reimburse himself by settling the payment from Carol. This act concludes the protocol, however Bob should consider offering an endpoint to Alice whereby she can *replace* the now-used payment hash and commit a new signature, that way Alice doesn't "run out" of unused payment hashes when her initial 1000 are used up. Also, Bob should offer an endpoint where Alice can replace her set of relays and commit a signature for the new set. This is wise because sometimes nostr relays stop working and if all of Alice's relays stop working without any way to replace them, senders won't be able to follow the protocol outlined in this document.

8. If you are implementing support for the zaplocker protocol I recommend you consider also implementing [the zap spec](https://github.com/nostr-protocol/nips/blob/master/57.md). It is not mandatory that zaplocker clients support zaps, but zaps are in the name, so consider it.
