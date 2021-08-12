# Sigchains
A sigchain is a chain of timestamped signatures. It can be used to issue and transfer tokens at the speed of the lightning network.

Suppose an oracle maintains a publicly viewable array -- which I will call a sigchain -- of partial pubkeys and timestamped signatures in this form: [{"partial pubkey": ["timestamp", "signature"]},{"partial pubkey": ["timestamp", "signature"]}...]. Suppose each signature is only valid for a string of data which is not provided by the oracle. Suppose the timestamp is 4 bytes, the partial pubkey is 12 bytes, and the signature is 64 bytes. In that case, each entry in the sigchain should be 80 bytes total: 4+12+64=80. Supposing such a publicly viewable sigchain exists and at least one person -- the oracle -- can add entries to it, suppose users send data to the oracle consisting of a partial pubkey and a signature, along with a lightning payment (at the oracle's discretion), and in return the oracle timestamps those key:sig pairs and immediately adds them to its public sigchain.

Suppose a user called Alice creates a pubkey and a message, signs her message with her pubkey, and then sends the oracle a lightning payment plus her pubkey and her signature (but not her message). If the oracle accepts the payment and adds Alice's data to the sigchain, Alice can send a copy of her pubkey and her message to Bob. Bob can use Alice's pubkey to look up the set of entries in the sigchain containing Alice's partial pubkey and her signature. If Bob finds a set of entries containing at least one match, Bob can extract the full pubkey from the signature of each entry in the returned set, verify that the pubkey matches the one Alice sent to Bob (or discard it if not), and then verify that Alice's pubkey really signed the message Alice sent to him (or discard it if not).

In this way, Alice should be able to quickly send Bob arbitrary messages which are attested to by a timestamping oracle in return for compensation paid via the lightning network.

# Censorship resistance

If a group of people wish to use a timestamping oracle without fear of censorship, it seems wise to keep their oracle in the dark as much as possible about the contents of their messages. Otherwise, the oracle might censor their entries and refuse to add them to the sigchain (even for a fee) if it does not want to attest to certain content.

In this sigchain system, censorship is unlikely if users do not reveal the contents of their messages to the oracle. If they do not reveal their messages to the oracle, it only sees key:sig pairs and a payment and is blind to the message that is attested to by the signature. Therefore, assuming that users only send the oracle their pubkeys and their signatures but not their message contents, the oracle should not have any reason to treat any messages differently from others.

However, that is not sufficient to avoid all censorship of the sigchain. For one thing, users might reveal the contents of their messages to the oracle, such as if the oracle is -- perhaps unbeknownst to them -- one of the people to whom they are sending a message. Also, the oracle might decide to censor an entry not based on the content of the associated message but based on the pubkey which created the entry. This might happen if oracles must, for legal reasons, abide by a blacklist of pubkeys and not attest to messages created by those pubkeys. Also, supposing the oracle ceases operations at some point, that is equivalent to censoring all of its users if no one can add to the sigchain anymore.

To ensure that sigchains are censorship resistant even in the face of these obstacles, there is an alternative way for users to add entries to the sigchain. Specifically, they can also take their key:sig pairs and create a bitcoin transaction where that data is entered in the op_return field. Suppose a user encodes the op_return data in this way: ["partial pubkey":"signature"] <-- that can substitute for sending the message to the oracle. A timestamp for the message can be inferred based on the block it is included in, thus eliminating the need to put a timestamp alongside the signature itself.

Software which scans the oracle's array in order to find out information about the sigchain can simultaneously scan bitcoin's blockchain for entries in that format and consider both sets of entries as constitutive of the sigchain. As long as the sigchain is defined not only as the set of key:sig pairs in the oracle's array but also as the set of key:sig pairs in op_returns on bitcoin's blockchain, the oracle cannot censor the sigchain but only deny users access to its own timestamping services. Effectively, there are two sets of people who can add entries to the sigchain: the oracle and bitcoin's miners. Either set can be "hired" to perform this service. If the oracle becomes censorious, bitcoin's miners can be relied upon to be less censorious (albeit possibly more expensive or slower).

# Immutability

There may be cases where sigchain software discourages pubkey reuse by checking if a pubkey has signed multiple messages and considers only one of them valid, specifically the one whose timestamp is the earliest. In such a case, it might seem possible to change a message after it has been timestamped as long as a second entry signed by the same pubkey can be inserted into the sigchain with a timestamp before the original.

One scenario to guard against is where bitcoin miners mine a block which adds an entry to the sigchain where the timestamp of the new block is in the past. In this context, it is useful to recall that there are limitations about how old a block's timestamp can realistically be. A block is invalid if its timestamp is older than the median timestamp of the past 11 blocks. In practice, no block has been mined whose timestamp was older than 7000 seconds in the past (which is a bit less than 2 hours). As a result, there is an easy way to reduce miners's ability to past-date a sigchain entry: if a sigchain entry appears in a bitcoin block, software which scans the sigchain for entries can add two hours to the timestamp of that block and use the adjusted timestamp for the sigchain entry. This procedure should be perfectly reproducible by all sigchain software and relatively harmless to users because users are not expected to use the op_return method of adding entries to the sigchain anyway except in extraordinary circumstances. Plus, the op_return method is already expected to be the slower option, so who cares if such transactions are even slower due to being post-dated for up to a few hours?

Preventing oracles from past-dating an entry in the sigchain is harder. I do not know a good way to absolutely prevent it but I can think of a way to discourage it. Suppose that every time an entry is added to the sigchain, the oracle must also use its own pubkey to create an entry on the sigchain. The oracle's entry must be a signed hash of its previous entry concatenated together with the other pubkey's entry. If the oracle does this every time, then every state transition in the sigchain will be attested to by the oracle in a way that makes past-dated insertions provably past-dated. In order to be past-dated, the new timestamp would necessarily belong in a position somewhere in the sigchain's past. If the oracle inserts it there, the chain of state attestations will need to be modified to include a new entry between two previous ones. Effectively, the oracle will have to redo all of the signatures to account for the new insertion, thus "re-signing" everything. Users who are affected by this state transition will then have public proof that the oracle resigned their transaction. They can therefore present this to the public as a fraud proof which may result in a reduction of that oracle's usage, harming the oracle. If the oracle stands to lose more by defecting users than it gains by past-dating a signature, it should avoid doing so. 

# Token issuance

Messages and sigchains can be used to issue and transfer assets at speeds roughly equivalent to bitcoin's lightning network. Suppose a user called Alice wants to issue 100 tokens. She can create an issuance message, hash it, sign the hash, and add her key:sig pair to the sigchain either via the oracle or via an op_return. An issuance message must have this format: "pubkey 1ef...50a received 100 tokens from an issuance transaction, the token id is 5d2...9a4." The recipient pubkey must sign issuance transactions and the token id must be globally unique for each asset.

Once Alice's signature is in the sigchain, Alice can prove to anyone that she issued 100 tokens by sending them her message and her pubkey. Other people can use the information provided by Alice and the sigchain to verify that 100 tokens were in fact issued and were sent to a pubkey. Alice could also issue more tokens of the same kind by creating a similar message again, thus inflating the supply of tokens. If her token is one where inflation is desirable, such as a usd-based stablecoin where more tokens are periodically issued when users pay to create them, that is well and good.

In cases where token issuances should be publicly auditable, there is a way to do that. Sigchains do not "care" what data users add in the signature field. They do not check whether it is a valid signature, for example, they only check that each entry's signature field contains only 64 bytes. If a public issuance is desirable, the issuer can use the signature field to add a publicly readable issuance transaction to the sigchain. I.e. the same message as above ("pubkey 1ef...50a received 100 tokens from an issuance transaction...") except inserted into the sigchain in a signature field in plaintext. This can be done via one transaction or multiple transactions which are to be parsed serially.

Later, I will explain how users who receive tokens can verify the provenance of those tokens before providing a receipt for the transaction. Users who desire to reject tokens whose provenance does not lead back to a public issuance can use wallets which refuse to provide a receipt for such transactions. Thus client side validation with sigchains is sufficient for ensuring that token issuance is auditable in cases where it is desirable for a token to have auditable supply.

# Token transfers

Suppose a user called Alice controls the pubkey 1ef...50a which has already received 100 tokens from an issuance transaction or another transfer transaction. Alice wants to transfer 50 tokens to another pubkey. She can create a transfer message, hash it, sign the hash, and add her key:sig pair to the sidechain either via the oracle or via an op_return. A transfer message must have this format: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a, the token id is 5d2...9a4." The sender's pubkey must sign transfer transactions. Clients only see them as valid if the sender sent tokens they previously received either via issuance transactions or through other transfers.

# Proof of provenance

Suppose a user called Alice wants to transfer 50 tokens to a user called Bob. Alice gets a pubkey from Bob (his pubkey is 4a1...990) and uses one of her own pubkeys (1ef...50a) to sign a transfer message. Then she sends Bob a copy of her pubkey, 1ef..50a, along with a copy of her transfer message. The transfer message is just like the one mentioned above: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a, the token id is 5d2...9a4," where pubkey 4a1...990 is Bob's pubkey. Bob can use Alice's pubkey to look up the set of entries in the sigchain containing Alice's partial pubkey and her signature. If Bob finds a set of entries containing at least one match, Bob can extract the full pubkey from the signature of each entry in the returned set and verify whether the pubkey matches the one Alice sent to Bob. If it does not match, he can discard that message and look for the next one, because it is not signed by Alice's pubkey, and therefore it is just as irrelevant as the other messages in the sigchain. (The pubkey extracted from a signature might not always match the partial pubkey prefixed to the signature for several reasons, one of which is that malicious actors can pay to add fake signatures to the sigchain prefixed with the partial pubkey of a user they don't like. It is important to check if the pubkey extracted from the signature matches the one provided to you by your counterparty lest you reject good transactions because of malicious actors.)

If none of the messages in the sigchain are signed by Alice's pubkey, Bob can refuse to give Alice a receipt for the transaction because she did not send him the tokens he wants -- she gave him a pubkey which never signed a transfer message or never got it attested on the sigchain. If, however, one of the signatures returned by his search "checks out," Bob can check if more than one checks out. If more than one checks out, Bob can check which one's timestamp is earliest and consider only that one to be a valid message. If every sigchain wallet abides by this rule, that should prevent double spends.

If some wallets do not abide by this rule, the users of those wallets will not have that protection against double spends. This major downside may encourage them to adopt wallets that only treat the earliest timestamped message by a single pubkey as valid.

Assuming Bob's wallet verifies that the earliest timestamped message from Alice's pubkey "checks out" in terms of having a signature that is attested on the sigchain, Bob's wallet is now at the part of the transaction where it should check the provenance of the tokens transferred to him. In order to do this, Alice's wallet must send Bob's wallet proof of provenance. Ideally, this should be done at the same time the token transfer message is sent. Perhaps Alice received her tokens directly from an issuance transaction; if so, she should send Bob the message which transferred her the tokens and the pubkey of the token issuer -- that is her proof of provenance.

Bob's wallet can use Alice's proof of provenance to check if the issuance transaction follows the rules his wallet follows. For example, was the issuance signed by a public key belonging to a company he trusts to legitimately issue tokens? Is the signature of the issuance attested to on the sigchain? Was it a publicly auditable issuance (assuming he doesn't want to accept tokens where the issuance was done privately and is therefore not auditable)? If the proof of provenance fails any of these checks, Bob's wallet can refuse to provide a receipt.

Perhaps Alice did not receive her tokens directly from an issuance transaction but from a prior transfer. If so, her proof of provenance will be longer. She will need to send Bob a trail of transfer messages and pubkeys leading back to the token issuance -- which could be a rather long proof of provenance. Once Bob's wallet has a complete proof of provenance from Alice, it can check everything out by ensuring that the tranfers and issuance transaction are attested on the sigchain. If any check fails, Bob's wallet can refuse to provide a receipt. In this way, Alice can only send Bob money and expect a receipt if she first validates that their wallets follow the same rules (such as by checking version numbers) and that her proofs of provenance check out before sending anything to the sigchain for attestation. This does impose a storage burden on Alice, who must store a train of pubkeys and messages in order to validly send anyone money and expect a receipt. But these storage costs should be low as long as pubkeys and signed messages are small.

Sigchain users can always see if a given pubkey ever double signed any messages on the sigchain and only accept the earliest messages from a given pubkey. They can also use the sigchain plus the proof of provenance to ensure that they are not receiving counterfeit tokens but only tokens that come from sources they trust. These protections should reduce the likelihood of counterfeiting and double spending to acceptable levels of trust minimization. All a user needs to do is verify that the sender has enough tokens to cover the expenditure (that is what the proof of provenance is for) and that they aren't accepting a late message (by only considering the earliest message signed by Alice's pubkey as valid). Given those two provable facts -- the sender has enough tokens to cover the expenditure and isn't double spending anything in this transaction (because this pubkey never signed anything before) -- there is no significant risk to the recipient.

# Token Change

In many of these examples I assumed Alice received 100 tokens at one of her pubkeys and later sent out 50. If wallets will never consider a second expenditure from her pubkey as valid, how can she ever get her money back? The answer is similar to how bitcoin works: when Alice sends money to Bob, she can send any "change" -- leftover funds not given to Bob -- to a new pubkey belonging to herself. Thus her transfer transaction would look like this: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a and pubkey 30c...bb4 received 50 tokens from 1ef...50a, the token id is 5d2...9a4," where 4a1...990 is one of Bob's pubkeys and 30c...bb4 is one of Alice's. If Alice is concerned that such a transaction will give Bob a way to check if she still has that money in the future (since he will see the transaction too), Alice can immediately send her money out again to yet another new pubkey. Since these transfers use lightning to pay the transfer fee and Bob will not receive this second transfer message, it should be a cheap way to restore Alice's privacy.

# Lightning speed

Issuing tokens or transferring tokens should not typically involve much waiting. The steps for issuing tokens are typically: validate some data, send it to an oracle, and pay a lightning payment. The issuance should be valid as soon as the oracle adds the entry to the sigchain, which shouldn't take more than a few seconds since it doesn't have to validate anything. Transferring tokens should also be very fast. It is the same procedure except now you have to also send a message to the recipient and wait for him to give you a receipt (after he looks up some signatures on the sigchain and then validates the data). Again, none of this should take a long time.

One exceptional case might be this: suppose an entry to the sigchain is made via an op_return instead of via the oracle. This might be necessary for a user whose pubkey is on some sort of blacklist. Hopefully, to avoid delays when buying their coffee, blacklisted individuals would transfer the tokens to a new pubkey via an op_return transaction well before they go to get coffee. These transactions are uncensorable and, unless the blacklisted person leaks the new pubkey to someone and it gets to the oracle, they should be able to use their funds again censor-free (and therefore at lightning speed). If they don't take that precaution, waiting for a confirmation will probably be necessary, unless the recipient is okay with receiving unconfirmed payments on the blockchain.

One might wonder if another exceptional case involves doublespend attempts. For example, suppose Alice adds an entry to the sigchain via the oracle which sends some tokens to Bob and simultaneously adds an entry via an op_return which sends the same tokens back to herself. Might Bob have to wait in this case? No. If Bob's wallet sees an entry in the sigchain via the oracle and another entry possibly made via an op_return (regardless of whether or not it is confirmed), all Bob's wallet needs to do is rely on the policy of "only the earliest one is valid." Op_return entries are post-dated by 4 hours, so as long as the oracle entry is confirmed now, that is the one Bob's wallet should consider valid.

It is also possible that the oracle is colluding with Alice and intends to drop Bob's entry and replace it with Alice's, or wait for the op_return to do that. This is a potential concern for Bob, who cannot know that this will happen in advance. If that happens, he will probably give a receipt and lose an item of value without getting any money for it, but at least he will have a proof of fraud by the oracle in the form of a state update that was signed by the oracle and then removed. He can use publish this on the internet to dissuade people from using the dishonest oracle. If the oracle can be trusted not to act maliciously in this way, the system is robust. 

# Smart contracts

Token transfer messages and token issuance messages can specify spending conditions that must be followed by token recipients. A field can be added to the transfer message saying something like: "The recipient can only spend this if they sign the transfer and disclose the preimage to this hash: 7ea...026, alternatively the sender can spend it if the timestamp of the expenditure is after time such-and-such." That would effectively put the funds in an htlc on the sigchain. A network of payment channels using htlcs could then be created that is basically a clone of the lightning network. Other spending conditions could also be enforced, developers could let their imaginations go wild.

If smart contracts are used, recipient wallets will need to understand the spending conditions of the tokens they receive and know what to do with them. If a wallet receives tokens but does not understand the spending conditions, it should refuse to provide a receipt. Perhaps the user can download a wallet which does recognize the spending conditions, import his private keys to that wallet, and then use the new wallet to issue a receipt.

A wallet will also need to understand the spending conditions of the ancestors of its tokens in case they restrict their descendants in ways which are not mentioned in the latest message. They can use the proof of provenance to check if any ancestor transactions created spending conditions which the current wallet does not understand.
