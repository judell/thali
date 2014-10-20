---
title: Building Thali on Top of Firefox
layout: default
---

# TLDR 

Thali could be entirely built using Firefox technologies and in so doing have a single code base that would literally run anywhere. But the complexity of doing so appears so high that for now it seems better to take a different approach.

# Investigation Results 

Firefox supports building add-ons in pure Javascript which are then portable, as is, across all Firefox platforms. They can do powerful things through the XPCOM api that exposes lots of fun functionality like raw sockets. But unfortunately many of the APIs we need are not exposed via XPCOM so we would have to build our own APIs using [js-ctypes](https://developer.mozilla.org/en-US/docs/Mozilla/js-ctypes) which isn’t easy but based on the [Network Security Services (NSS) library](https://developer.mozilla.org/en-US/docs/NSS) and given the example of [https://wiki.mozilla.org/Firefox_Sync Firefox Sync](https://wiki.mozilla.org/Firefox_Sync Firefox Sync) (which is written as a Javascript extension that does its crypto by calling down to NSS via js-ctype) it’s doable. Then, on top of that, we would need to be able to run the server portion as a long living background process with no UX. There is a way to do this in Firefox using a 'headless' version of Firefox called xpcshell. But I was warned on the Firefox IRC channel that not many people really understand xpcshell and most of them don't have time to hang out on IRC so I would be on my own. Note, btw, that once all of this is done I would still need to find a HTTP server and a HTTP client, written in Javascript, to sit on top of the js-ctype APIs. There are, limited, attempts at least at HTTP server, I didn't find a HTTP client but I'm sure it's out there. Writing both isn't brain surgery but it just becomes another set of non-trivial code that I have to write, test and maintain.

The end result is that what Thali needs is doable in Firefox. But the cost and complexity seems to high.

# Required Functionality 

The following is the core functionality we are depending on Java for at the moment.

* Generate RSA 2048 or better yet 4096 public/private key and put it in a PKCS12 file
* Generate X.509 cert 
* Parse X.509 Cert
* Validate X.509 cert chain
* Open incoming SSL connection using self signed cert (or self signed cert chain) & being able to accept any cert from the client and then get the client cert for further analysis
* Open outgoing SSL connection and be able to specify what cert you expect the server to present (or chain to) and specify what cert you want to present
* HTTP Server that can run over the ssl connection  [http://mxr.mozilla.org/mozilla-central/source/dom/network/tests/unit/test_tcpsocket.js](http://mxr.mozilla.org/mozilla-central/source/dom/network/tests/unit/test_tcpsocket.js) shows a test for opening both a client and server socket but it doesn’t use SSL which is actually the subject of two open bugs.
* HTTP client that can run over the ssl connection
* SSDP or ZeroConf support and/or the ability to send and receive UDP

# Useful resources 

[http://mxr.mozilla.org/mozilla-central/ident](http://mxr.mozilla.org/mozilla-central/ident) - Provides a search interface over Mozilla's source code. I've been using it to find functionality.

# Thanks 

Much thanks (or, depending on how things turn out, curses) to Oleg Romashin and Scott Blomquist for pointing this path out. And to John-Galt (who ever you are) from #addons IRC on Mozilla for pointing out NSS.

A client MUST NOT establish a connection using a httpkey URL if it does not recognize/support the key type in the URL.

After the TLS connection is established then further HTTP processing occurs as normal with the exception listed below. The distinction then between httpkey and http URL processing is just in the TLS connection setup.

The request-target value in the request-line MUST be the fully qualified httpkey URL.

# Example 

<pre>
 httpkey://ku7mzposmlr5343f.onion:9898/rsapublickey:65537.229123329158186784221508160085
 675953045725302707662388599223430327916129668245579329476599603333511533884351582846663
 092488251759749119644311701416239069310408566641309818421770606018830931913117414055303
 531803349718235807503444353141974738331818988420105667389930752590018085664633480275231
 415428099264983353242738028996077248314140787293700965179586583463742702056210712633617
 796830512423632872229877354180111877712047188831455202520895028152738438935287108085195
 264731128747745618511381018968060137975982778955390343303280948772760845339675078319105
 23283288815798592996543256860992426377095371666673172691091922277/addressbook
</pre>

This httpkey URL is used to talk to a Thali Device Hub's addressbook database. Please note that the introduced white space is just for readability purposes.

## Q&A 

## Is this spec done? 

No. We need to go through the HTTP and HTTPS specs with a microscope and figure out all the places that things are underspecified. For example, if someone connects via a httpkey URL should they accept a redirect to a non-httpkey URL? What if the redirect is to a httpkey URL but with a different key? Etc.

## Don’t we need requirements for the accepted configuration parameters for TLS? 

Yes. I intend to expand the spec to identify allowable algorithms, key sizes, etc. But right now I just want to get something running.

## What should go into the request line on a HTTP request using httpkey?  

So I would suspect that the right answer is the fully qualified httpkey URL (and yes, [that is legal](http://tools.ietf.org/html/draft-ietf-httpbis-p1-messaging-26#section-5.3)). But right now all my code actually turns the URL into a HTTPS URL locally and puts that through the stack (after I've fixed up the socket underneath to support mutual SSL auth and Tor) because, well, it's easier. But at some point I should do the right thing.

## What about the connect method? 

Connect doesn’t really apply here since this is about how to process a httpkey URL which requires establishing a TLS connection from the get go. That is intentional since Thali’s philosophy is that all connections should always be secure. Connect is when you start with an unencrypted connection and want to negotiate up but we don’t support that.

And yes, I realize Connect can be used for other purposes. Once a httpkey connection is established it’s really just HTTP so you can do whatever you want.

## How is this different than HTTPS? 

HTTPS is fundamentally about binding a DNS name to a public key through the attestation of a set of certificate authorities. Httpkey is about validating that a presented key is the expected key. There is no DNS name and no certificate authorities. But in both cases once the security is done what one is left with is a HTTP connection.

## Why don't you use a hash of the key rather than key? 

Just one more thing to get wrong or to provide a particular security attack against. The main advantage, obviously, for a hash is brevity. But even a hashed key is unreadable and untype-able. So to heck with it, let’s just simplify things and put the whole key in. In the old days there were additional concerns about URL sizes but now that whole files get moved around in URLs that isn’t the problem it used to be.

## Why don't you use the cert instead of the key? 

Because we aren’t validating a cert, we are validating a key. A cert is a way for some authority to make assertions about a key. And that is useful in httpkey when we deal with chaining but the thing being validate is the key, not the cert. So that’s why the key is in the URL.

## What about key expiration? Or key revocation? 

Experience over the last several decades argues that key revocation just doesn’t work. Nobody checks the revocation lists. Checking them on each request is too expensive in terms of latency (since in theory you shouldn’t try your connection until the revocation list is validated) and the behavior when the list endpoint isn’t available gets nasty (do you refuse the connection?). So generally revocation lists are considered a failure. Expiration is much more useful. I suspect that for every time expiration successfully kept a key from being inappropriately used there are 1,000,000 (and yes, I mean that many) cases where expiration just made an otherwise fine connection fail.

So my general feeling is that one got a httpkey URL from a context. To the extent that revocation or expiration matters then that should be handled in that source context, not by trying to turn the URL into a mini-file format.

## Why is cert chaining supported at all? Wouldn’t it be simpler to leave it out? 

It turns out to have really nice security properties. For example, a properly paranoid person would never have their root key on the Internet. Instead they would use an air gapped computer that uses a limited communication channel (infrared? Camera? Nfc?) to issue keys to devices. The identity key is therefore never directly on the net and we now have set up a way to limit the potential damage if a device (or other trusted entity) is compromised. The assumption being that Thali or some other context from which the httpkey comes has a way to push out information about revocations. So we like cert chains.

## What if the desired key is in the middle instead of the end of a chain of certs? 

Right now the spec just says that the key being sought for has to show up somewhere in the chain, it doesn’t say it has to be the root. I’m still not sure if that’s a bug or a feature.

## What about YURLS? 

There really is nothing new under the sun. I described the idea for the httpkey URL scheme to a coworker (Hey [Julien](http://blog.monstuff.com/)!) and he immediately pointed to me to [YURLs](http://www.waterken.com/dev/YURL/). To be fair these ideas go back even farther to places like SPKI and SDSI.

## Doesn't TLS leak user identities?

I am not a TLS expert and I don't play one one TV but it seems pretty obvious that the answer is, unfortunately, YES.

The problem is that, near as I can tell, the TLS handshake sends both the client and server certs in the clear. As a consequence if someone is observing either end of the connection they can, just by observing the certs used, determine the identity of both the client and server.

And yes, this is really bad because it means that a passive observer can just look at the connections (without having to actually break any crypto) and determine who is talking to whom.

Interestingly enough this bug doesn't really matter when we are talking over Tor to a Tor hidden service. The reason is that Tor hidden services end to end encrypt all messages. So the handshake is only done over an encrypted connection between the client and server.

So where this is a real problem in situations where we are talking directly, not via Tor. This should really only happen in ad-hoc networking scenarios where the Internet is simply not available. The problem still sucks though because we should be able to communicate directly (outside of Tor) without having to advertise our identity all of the damn place!

I have broken this issue into two scenarios.

<dl>
<dt> Public Discovery</dt>
<dd> In this scenario a Thali user is on the local ad-hoc network and wants to let people know they are there so they can connect.</dd>
</dl>

In this scenario there is obviously no way to avoid advertising the server key (it's inherent to broadcast based discovery) but do we also have to expose the client key? There is a well established pattern for handling this - renegotiation. 

1. The client opens a normal HTTPS connection to the server and just validates the server's key

1. The server or client force a renegotiation and now mutual auth is required.

Because of how TLS is designed the renegotiation handshake happens over the existing encrypted connection and so the client's identity is protected.

<dl>
<dt> Private Communication</dt>
<dd> A Thali client and server somehow find each other on the local network and want to establish a connection without leaking any identity</dd>
</dl>

The 'somehow' part is important. Imagine that peer A gives a key to everyone its willing to talk to (this is a pre-established set). When A gets on an ad-hoc network it takes a random initialization vector plus its identity and encrypts them with the key and advertises the result (e.g. broadcasts it on the local network). Since most peers have a small set of other peers they would accept on the ad-hoc connection it is computationally feasible for a peer to intercept the advertised ID and try it against all the keys of all the peers they have. If it's a recognized peer and they want to talk then it knows the peer's IP address. But an observer who isn't privy to the key just sees junk.

So imagine peer A advertises their location using the above strategy. Peer B sees the broadcast and wants to connect. So now Peer B opens a TLS connection to the address that broadcast the message (e.g. Peer A). As part of its handshake Peer B uses SNI to send in an encrypted token with a unique previously agreed symmetric key that tells Peer A the identity of Peer B. Peer A tests the SNI against all the peer keys it has to see which one it matches. If it matches none then it kills the connection. Otherwise it now knows who the caller is claiming to be. At that point when negotiating the new connection peer B returns the same ephemeral key but this time chained up to its full chain to its true identity. It is willing to expose its identity because the SNI token reasonably proves that peer B is allowed to know peer A's identity. Peer B gets the root chain, confirms its the same ephemeral key at the leaf (needed to prevent man in the middle attacks) and then returns its client cert. Everything after the renegotiation is inside an encrypted tunnel and so not visible.

So we *can* work around this. But it looks like the TLS 1.3 working group is working on this problem so hopefully they will solve it for us.

## Wait... does this mean that clients must always use Tor? 

In other words, are clients always required to have a client Tor Tunnel?

No. In theory I believe it's actually possible to talk to a Tor hidden service with just a single hop client tunnel (e.g. just the rendezvous proxy). And when talking to non-onion endpoints one can use Tor or not. The httpkey URL type doesn't specify the required behavior. 

But Thali does. 

We use Tor whenever possible. 

But there are times when a user might legitimately choose not to use Tor. Either they are trying for a non-routable address (e.g. local network only) or they want better performance and are willing to give up some privacy to get it. But the choice for the client to use its own Tor tunnel is not something that the httpkey URL spec has anything to say about. The job of a URL is to specify where the resource is and that's it. The choice of using a client Tor tunnel doesn't apply to that problem.

## What's up with the prohibition on resolving .onion addresses outside of Tor? 

The .onion address looks like a domain name that has the root domain '.onion'. This is something you can pass to a DNS resolver and try to resolve. It will fail (at least until .onion is standardized) but even the act of looking up the address leaks information to the DNS server and if DNSSEC isn't being used, to whomever is monitoring look ups. That's why we can only look up .onion addresses via the Tor infrastructure.

## What about other systems like I2P? 

There is no reason we can't support I2P. It's not clear if we would do this through a new URL type or through a domain hack. Given I2P's nature I would suspect the former but regardless we can get there from here. I actually had a lovely chat with the I2P developers and it turns out that currently they require UDP access to function and that is a show stopper for us since we need to run behind stateful firewalls that only support outgoing TCP on port 80 & 443. But there is nothing fundamental in I2P that stops it from running on port 443 over TCP. Furthermore there is nothing stopping us once we hit our min bar from putting in I2P support anyway.

## What about the Tor hidden service's authorization features?

Sigh... so um... this is kind of embarrassing. But I actually don't really understand the features described in sections 2.1 and 2.2 of [rend-spec](https://gitweb.torproject.org/torspec.git/blob/HEAD:/rend-spec.txt). I need to ask for help from the Tor group.

I *think* that in section 2.1 the idea is that you give each user a cookie. The user then goes to the descriptor database and looks up the desired endpoint. In the descriptor at the endpoint will be a list of IDs followed by a session key that has been encrypted with the cookie. So the caller can generate which ID matches them and then pull out the encrypted value, decrypt it with the cookie, thus getting the session key and use that session key to decrypt the list of endpoints where the service can be reached. Due to limits on the size of descriptors only 512 cookies can be supported. The upside is that 512 is a reasonable big number, especially for most Thali scenarios. The downside is that because there is exactly one descriptor entry for all 512 entries and it is associated with the site's public key it's possible for attackers (or formerly authorized users) to observe who is looking up the service's information.

Section 2.2 actually seems to generate a new public key associated with a particular client and a cookie, both are given to the client. The service then generates up to 16 descriptors, each associated with the public key for one of the clients. The endpoints are then encrypted with the cookie. The point then is that each client of the service ends up with a completely different descriptor entry indexed with a different public key and encrypted with a different cookie. This makes it theoretically impossible for anyone to observe who is looking the service up. But again, the system only supports setting up 16 of these special descriptors.

I suspect both authorization mechanisms have a place in Thali. But nothing like the client key or cookie should ever appear in a URL! These are secrets. So they will need to be communicated out of band. Eventually we'll need to define how that communication occurs and put in the software to support configuring everything on the client and server side. But all of those issues are actually out of scope for this spec.

## Does it really matter that Tor doesn't support cert chaining for identifying hidden services?

Because of the lack of cert chaining it's not a good idea for a user to use their public key as their Tor hidden service key. See above for the details on that.

So this means that Thali users will need two keys. Their personal public key and their Tor hidden service key.

This also means that to communicate with a user one has to have both keys (there are some exceptions in local networking scenarios but I'm going to ignore them for now).

So this means that anytime a user wants to advertise their location they need to provide both keys.

The problem is that the Tor hidden service key needs to be regularly rotated for security reasons because it is actively used. Now we can have key roll over where the user connects to both the old and new Tor hidden service keys for some period in order to tell people about the change. But what happens when there is a systemic failure? We have a heartbleed style attack where we know that all our Tor hidden service keys are compromised. Now everyone has to change their keys at once and can't risk using the old keys. How do people find their friends new Tor hidden service keys?

With cert chaining this could be dealt with because the root of the chain will be executed completely offline or at least in a physically separate process so the probability of that being compromised is a lot less. If we have a compromise of the Thali process that is connected to the network that would only compromise a non-root cert which could be replaced easily without anyone having to change anything. But to be fair this benefit only accrues if the flaw doesn't compromise the root cert. With root certs likely hanging out on devices it doesn't take much imagination to create a scenario where root certs are also compromised. So while chaining is useful it is not a magical elixir.

The other problem is longevity. I've had the same email address for more than a decade. Anyone who can find it can email me. We would like the same experience for those who want it in Thali but how do we make that work if the Tor hidden service key is constantly changing? Finding my public key isn't enough. You need the current Tor hidden service key as well. If we had cert chaining then we could actually advertise an endpoint in the descriptor directory using my public key directly. But remember we NEVER want to have the holder of the private key talk to anyone over the network. So we have to be able to get a descriptor into the directory with my public key without using my public key. So we need chain support.

If we can't get cert chain support then we'll need to emulate it through another mechanism. I can imagine us using something like [freenet](https://freenetproject.org/) but getting it to support cert chains so someone can publish something indexed to key X using key Y where they can present a chain from Y to X. But this is just moving the problem around, not solving it and anyway I hate to introduce a whole other project though so my hope is that we can just get cert chain support in Tor directories.

## Isn't it good that you can't use the Thali user key as the address? Otherwise everyone could observe traffic at the directories related to you

It is a feature that one can choose to have a Tor key that is different than one's personal key because it does make it more difficult to look at who wants to talk to you by monitoring the look up directories. But in the case mentioned above where one wants to be discoverable by previously unknown principals then alas directory sniffing is the price.

So we want to enable folks who want a public identity to be able to do so but also support those who want to stay strictly private to do that as well. So having two different keys, one for the person and one for Tor is a feature in some cases.

## How do you revoke an existing onion address and publish a new one?!?

There are two scenarios here. One is a roll over. Presumably in that case the TDH will listen on both the new and old address and let anyone who connects to the old address know that they should switch to the new address. One way to 'let them know' is to push a sync to the connector letting them know about the update.

Then there is the 'oy! My Tor key is compromised!!!' scenario. In that case the TDH will have to push a synch to all of their friends letting them know about the change. The synch will work because it's the Tor key and not the root key that was compromised.

And yes, per the previous questions, if everyone's Tor key is compromised (again, think heartbeat) then we are in deep trouble because we have no way to securely connect to folks or even find them. Again, why chaining would be useful. It would at least address the scenario where the Tor key and not the root key is compromised.
