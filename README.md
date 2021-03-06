# An RFC for provable mailing list subscriptions

This document serves as a discussion area for the aims of this project - it is not the project itself.

## The problem
Mailing list subscriptions are typically created using a "double-opt-in" process, where a potential subscriber requests a subscription (the first "opt-in") and is sent an email message containing an HTTP confirmation link, which they click to verify and approve the subscription (the second "opt-in", hence "double-opt-in"). The second opt-in is required as the first alone does not provide sufficient evidence that the address provided really did send the request - it could be spoofed. During this process various pieces of evidence can be collected in order to prove that this transaction took place. These are typically:

* The email address that requested the subscription
* A timestamp
* The user-agent string of the browser used to click the link
* The IP address that clicked the link (can help to prove provenance of the click - e.g. it can usually be narrowed to at least the ISP and country, if not an individual)
* The URL of the page that the request originated from (e.g. a sign-up page)
* A copy of the verification message that was sent to the subscriber (a requirement under German law since 2014)

While these data points contribute to a reasonable circumstantial argument that a given subscription is legitimate, all of this data is forgeable, and almost none of it is exposed to external parties that might want/need to know in order to process messages sent out under the list subscription. This is particularly relevant to large ISPs such as gmail, hotmail, yahoo, AOL etc who have a very difficult time filtering messages accurately. Correspondingly, ESPs doing the sending face a constant battle to have their messages accepted consistently without spam filtering, delivery deferral, rate limiting, and other countermeaures designed to combat spammers rather than legitimate volume deliveries.

A key problem with this data is that it is not portable - if a mailing list is transferred between ESPs, it usually omits all of this information, and it's unsual for ESPs to even have a mechanism for importing this data, despite its importance. It's also bulky and inconvenient. This has been a key contributing factor to the rise of unscrupulous list sellers, and a corresponding rise in spam.

There is a potential legal conflict: ESPs are required to retain this data to "prove" opt-in, but at the same time it contains personal data (particularly IP and email address) which are subject to GDPR data protection rules.

It may be useful to make this information also work for portable suppression lists, in much the same way as [the ESPC's standard for hash-based suppression lists](http://www.espcoalition.org/legal-industry-member-resources/industry-resources/suppression).

**We need a better way of proving that verified subscriptions have taken place in a way that is also portable across ESPs. Ideally, this information would also be used by ISPs to whitelist messages**.

## Related standards
We already have standards that help to prove various other aspects relating to email delivery:
* TLS & DKIM prove that messages have not been tampered with in transit
* SPF & DKIM prove that messages originate from legitimate sources for a domain
* DANE & DNSSEC prove that DNS entries are trusted
* DMARC to tell receivers what to do when these rules are broken

## Requirements
The solution must be:
* Integrated into the double-opt-in process, presenting little friction for subscribers
* Easy for subscribers to withdraw consent in a persistent manner
* Not contain personal data
* Cryptographically verifiable and resistant to forgery (i.e. not reliant on circumstantial evidence)
* Small and relatively simple (e.g. key elements should fit in an email or HTTP header)
* Portable across ESPs, so that proof obtained by one ESP remains valid if moved to another ESP.
* Not require significant changes to email infrastructure (because that won't happen!)

## Constraints
* If a proof is going to be portable, it must be centred on the email address of the subscriber, not the sender.
* To be verifiable by a receiver, it must be done using public information, for example making use of DKIM public keys.
* It can't be dependent on ESP private data - though there could perhaps be a different kind of identifier used for data transfers than for message sending, allowing the proof to be re-keyed to the new sender.

## Random discussion
This problem was partly solved in the distant past, in the days before the web; confirmation was done by replying to the subscription email message also by email, rather than using an HTTP link. Though it was not used at the time, this could be turned into a far more concrete proof nowadays because the return message could be provable via mechanisms such as DKIM. The problem with that is that not everyone implements DKIM, and it's much less convenient than a simple HTTP link. An added advantage is that a receiver replying *to* an address will probably mean that other messages *from* that address will be whitelisted in future.

DKIM is currently only used for signing email - but its authors had the foresight not to restrict it to that. The [RFC6376 service type field](https://tools.ietf.org/html/rfc6376#section-7.8) that's placed in DKIM headers can be extended to use the signature for other purposes, like this idea.

### Receiver signing service
A key problem with the currrent system is that verification is performed by the sender, and that means it's easy for them to forge evidence. Perhaps a better approach would be to get the receiving system to offer a confirmation service, maybe something that could live in the `/.well-known` space like web-based ACME verifications do. If I'm an ESP sending an opt-in confirmation to some user `@gmail.com`, the verification link could point at a URL on the host's MX, something like: 

```
https://smtp.gmail.com/.well-known/list-subscribe-confirm?id=abc123&sender_callback=https%3A%2F%2Fesp.example.com%2Fconfirm%5Fsubscribe.php`
```

which provides a callback service to the ESP, but also gives the receiver an opportunity to:

1. know that the subscription was accepted so that it can allow-list the source in spam filters
1. sign the response so that the ESP has a verifiable receipt that it can use to prove the subscription.

In order to prevent abuse of this callback (e.g. for SSRF), the sender could perhaps set some other DNS record that can be used to verify callback addresses. This signed response can also provide verifiable proof of subscription when transferring lists between ESPs.
