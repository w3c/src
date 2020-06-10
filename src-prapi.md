# Proposed Architecture for SRC through Payment Request API 

This is a proposed architecture for SRC through Payment Request API. 

* It is informed by discussion of [UX assumptions and requirements](https://github.com/w3c/src/wiki/UX-Assumptions-and-Requirements)
* See [detailed flow diagrams](https://github.com/w3c/src/wiki/SRC-Common-Payment-Handler-Concept) aligned with this architecture.
* This document does not focus on the [SRC data model](https://github.com/w3c/src/wiki). We anticipate that any future payment method definition will nearly entirely draw on the EMVCo data model.

Status: This document does not reflect consensus; it is for discussion.

In this document we refer to current implementation of APIs in Chromium browsers. These references are for illustrative purposes only and not meant to imply that all browsers must provide the same user experience.

# Value Proposition for SRC through Payment Request

SRC implementations exist that do not use Payment Request. Here are the primary usability and security benefits of a Payment Request (and payment handler) approach.

## Usability

* **Streamlined user experience**. Having a common payment handler enables automation to streamline transactions.
   * How it works: A common payment handler enables the browser to "skip-the-sheet" so the user does not need to select a wallet. Instead, the user is presented with enrolled cards to quickly complete a transaction, or the ability to add a card.
* **Harmonized experience across sites**. A common payment handler used across the Web is likely to foster trust and be easier to use than a series of user experiences that may differ from site to site.
* **Preserves checkout context**. A design goal is to keep the user close to the merchant site, so that the checkout context remains visible.
  * How it works: In Chromium browsers, the payment handler can open a modal window that remains close to the merchant checkout page. This makes it less disorienting than a redirect.
* **Less confusing, especially on mobile**. If the browser opens a pop-up and the user clicks on the window or changes tabs, the browser hides the pop-up behind the window, which can confuse the user. This is especially true on a mobile device because popups open as new tabs. In addition, because it can be difficult to keep track of which tab a popup belongs to, this can be particularly confusing in a scenario where the user has opened a large number of tabs while comparison shopping. Pop-ups are also generally locked down and difficult to invoke reliably due to the measures introduced by browsers to counter their abuse. Finally, some mobile browsers limit the number of tabs that can be opened at any one time (leading to potential data loss). 
  * How it works: Payment handlers, as a new capability in browsers, can provide a better experience than pop-up windows. The payment-handler-in-a-modal window approach enables the browser to eliminate confusion resulting from user interactions.
* **Persistence**. Browser vendors are changing behaviors around storage and access from third-party origins. This will make it more difficult for developers to provide a streamlined user experience across the Web (e.g., because it will complicate session management). 
  * How it works: We expect trusted payment handlers to have easier access to first-party storage access (with user consent). This is likely to make it easier to maintain state as the user makes payments across the Web.

## Security

* **Phishing-resistant**. Unlabeled windows opened by merchants or their PSPs can create phishing risks.
  * How it works: The modal window approach improves security because the browser displays the payment handler origin, making phishing more difficult.
* **Better risk assessment**. Backend risk engines may rely in part on information stored in the browser. Payment handlers may provide more reliable information.
  * How it works: As mentioned above, we expect trusted payment handlers to have easier access to first-party storage access. This is likely to make it easier to maintain state as the user makes payments across the Web, and this should improve data used by risk engines.
* **Optimizations for authentication**. we continue to develop APIs such as Payment Request and Web Authentication so that they work effectively in combination to reduce friction and improve security. Our goal is to provide powerful browser capabilities for the payments ecosystem that are superior than what can be achieved by using other parts of the Web toolkit out of the box.

# Terminology

## SRC Terminology
* For authoritative explanations of SRC terms, please see the [EMV® Secure Remote Commerce Specification](https://www.emvco.com/emv-technologies/src/).
* In this document we mention in particular these SRC Roles (borrowing descriptions from the SRC specification):
   * SRC System: A technical platform that securely facilitates remote card payments between Consumers, Digital Payment Applications, SRC Initiators, SRC Participating Issuers and Digital Card Facilitators on behalf of one or more SRC Programmes. A Digital Payment Application is any payment-enabled application that facilitates a payment between the acceptance environment and a Customer using a Card within an SRC ecosystem.
   * SRC Initiator (SRC-I): A role that supports Checkout and/or the secure retrieval of Payment Data from the SRC System on behalf of a Digital Payment Application.
   * Digital Card Facilitator (DCF): A role within the SRC ecosystem that provides a Consumer with access to one or more Digital Cards and facilitates the checkout experience.

## Web Payments Terminology

* A payment method is a data model. A payment method specification describes the input and output data (and some other things) to enable a payment to happen.
* A payment app is a piece of software that implements a payment method (or more than one).
* There are three ways to implement a payment app: built into the browser, native mobile, and Web-based. The Payment Handler API describes how Web-based payment apps communicate with the browser. Thus, we also use the phrase "payment handler" to refer to a Web-based payment app.

Note that a browser can play multiple roles in this ecosystem:

* It implements Payment Request API. It has certain jobs in that role such as figuring out which payment apps can be used for the transaction.
* It implements Payment Handler API to support Web-based payment apps.
* It could also act as a payment app itself for some select payment methods.

# Architecture 

## Identification

* There will be one Payment Method Identifier (PMI) per SRC system. Each PMI will have a different origin.
* The Payment Method Manifest associated with each PMI will refer to the same default payment app, which will live at its own origin. We refer to this as the Common Payment Handler.
   * **Note**: This proposed PMI architecture requires a change in Chrome to allow for a manifest at one origin to refer to a default payment app at another origin for JIT registration. If that change does not happen, we can still achieve our goals (JIT registration, skip-the-sheet, no wallet selector, SRC-system specific data blobs) by adding one more PMI to the architecture (that of the default payment app origin) that would appear in the call to Payment Request. 
   
FOR DISCUSSION: If a particular browser supports a payment method with a built-in payment handler, how does that interact with a payment method manifest from the owner of the payment method? Should it "just work" or should the payment method owner have some mechanism (e.g., via the payment manifest) to authorize built-in payment handler support from some or "all" browsers?

## Payment Handlers

We focus in this proposal on two ways a payment handler might implement an SRC payment method:

* Web-based third-party "Common Payment Handler" using Payment Handler API
* Built into the browser
* Native support of payment method in the browser

### Third-party "Common Payment Handler"

* Having a Common Payment Handler:
   * Facilitates just-in-time installation
   * Eliminates the need for users to select from among multiple digital wallets, thus enabling skip-the-sheet.
* The Common Payment Handler will have limited functionality:
   * Redirecting the user to an SRC-I according to data provided by the merchant in the call to Payment Request.
   * Storing information (provided by the SRC-I) to facilitate recognition of the user during future transactions.
* SRC-Is will be responsible for other user interactions with SRC systems (enrollment of cards, authentication, storage of contact and shipping addresses, etc.).
* The Common Payment Handler is expected to support delegation of the merchant's request for shipping and contact information. 
* The Common Payment Handler is expected to support methods such as changeShippingAddress and changeShippingOption in conjunction with the SRC-I so that the merchant can update the total based on address selection.

#### New user journey

In this scenario, the user has no SRC identity and no previously installed payment handlers.

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. There is no installed Common Payment Handler, so the reply is "false".
* The user pushes the "buy" button, which calls show().
* The browser displays the Common Payment Handler for just-in-time installation. 
* The user selects the Common Payment Handler, which redirects the user to an SRC-I of the merchant's choosing, according to data provided by the merchant in the call to Payment Request.
   * **Note**: To accommodate some use cases we may want the data regarding the SRC-I (e.g., a URL) to be optional, and (for example) the implementation of the payment method would choose an SRC-I.
* The SRC-I authenticates the user and stores this information for future transactions. The user enrolls a card or selects a previously enrolled card to pay.
* The SRC-I returns data (including data to identify the user in future transactions) to the Common Payment Handler. The Common Payment Handler stores information in the browser to facilitate recognition in future transactions.
* The Common Payment Handler returns the payload to the browser.
* The browser returns the payload to the merchant via Payment Request.

#### Returning recognized user journey

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. The Common Payment Handler accesses the information it previously stored about the recognized user and returns "true".
* The user pushes the "buy" button, which calls show().
* The browser skips the sheet and launches the Common Payment Handler.
* The Common Payment Handler redirects the user to an SRC-I of the merchant's choosing, according to data provided by the merchant in the call to Payment Request.
* The SRC-I returns data to the Common Payment Handler. 
* The Common Payment Handler returns the payload to the browser.
* The browser returns the payload to the merchant via Payment Request.

#### Returning unrecognized user journey

* The flow here is the same as for the first-time user, except there is no need to re-install the Common payment handler.

#### Uninstalling the payment handler

There may be times when the user wishes to uninstall a payment handler from the browser. We anticipate this will be done through browser UX.

There may be other times when a card becomes invalid and the SRC system (or payment handler) may wish to uninstall the Common Payment Handler.

There are two ways to remove a payment handler from the browser:

* Use a self-destroying service worker; see [example](https://github.com/NekR/self-destroying-sw)
* Clear the payment handler's instruments with ServiceWorkerRegistration.paymentManager.instruments().clear(). The service worker will remain in the browser, but can no longer be used as a payment handler.

The first approach provides a cleaner uninstallation than the second.

### Payment Handler Built Into the Browser

There might be multiple architectures for built-in payment method support, such as:

* The browser plays the SRC role of the DCF. By doing so, the browser removes the need for the merchant-specified SRC-I to render the card list. This approach has several advantages:
   * The UX is built into the browser, reducing the burden on PSPs would would otherwise have to integrate into the Common Payment Handler.
   * Where user identity is known to the browser, it may be possible to create longer lasting sessions.
   * The user experience would likely be consistent across all card networks.
* The browser itself implements a container analogous to the Common Payment Handler, and thus houses SRC-I code.
* The browser interacts with another piece of software (e.g., built into the operating system) through a proprietary mechanism. That piece of software is a container analogous to the Common Payment Handler, and thus houses SRC-I code.

The journeys below are described in a way that is independent of architectural choice (or at least strives to be). The browser is acting in multiple roles:

* Implementation of payment request
* Payment handler for SRC
* Possibly other roles (e.g., SRC-I).

#### New user journey
In this scenario, the user has no SRC identity and no previously installed payment handlers.

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. There is no registered instrument, so the reply is "false".
* The user pushes the "buy" button, which calls show().
* The browser displays the (built-in) payment handler, which invokes the SRC-I.
* The SRC-I authenticates the user. The user enrolls a card or selects a previously enrolled card to pay.
* The SRC-I returns data (including data to identify the user in future transactions) to the payment handler. The payment handler stores this information to facilitate recognition in future transactions.
* Functionally, the (built-in) payment handler returns the payload to the browser.
* The browser returns the payload to the merchant via Payment Request.

#### Returning recognized user journey

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. The browser accesses the information it previously stored about the recognized user and returns "true".
* The user pushes the "buy" button, which calls show().
* The browser displays the (built-in) payment handler, which invokes the SRC-I.
* The user chooses a card from the SRC-I.
* The SRC-I returns data to the (built-in) payment handler.
* Functionally, the (built-in) payment handler returns the payload to the browser.
* The browser returns the payload to the merchant via Payment Request.

#### Returning unrecognized user journey

* The flow here is the same as for the first-time user.

#### Uninstalling the payment handler

There may be times when the user wishes to clear payment data from the browser. We anticipate this will be done through browser UX.

### Native Support for Payment Method Into the Browser

<em>Note: It is possible that this option and the previous one can be combined; we are working on the relationship between these two sections.</em>

Key benefits of this approach are:
* Merchants or PSP can call existing Payment Request API (for SRC common payment handler approach or in combination with other payment methods).
* PSPs do not need to do any custom work for rendering UX for browsers that support this capability. 
   * Note: PSPs may need to develop custom JS integration with SRC Systems on browsers that do not have this OR do not have common payment handler. 
* User Experience is reliable - given lack of reliance on any local storage - and consistent across all card brands. 

Key downside of this approach: This requires browsers to create support for native payment method.

Key tenets of the architecture are listed below. 
* Merchant or their PSP invoke the payment request for SRC as a payment method identifiers
* The browser provides native support of payment method
   * Note: If browser does not support payment method natively, fallback approach is to rely on common payment handler.
* Browser provides end to end user interaction, while connecting with SRC APIs to facilitate the UX and payload request.
* SRC System provides APIs to facilitate cardholder verification as well as serving the merchant specific requests

Following key user journeys need to be supported in this scenario:

#### First time use of card on browser 

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. 
There is no registered instrument, so the reply is "false".
* The user pushes the "buy" button, which calls show().
* The browser facilitates end to end UX while integrating with SRC System(s)
   * **Note**: UX includes card entry, possible challenge based on SRC System response, as well as confirmation UX for user to confirm (e.g. via local authentication)
* On user confirmation of SRC transaction, browser sends the request to SRC system. SRC system sends opaque payload (checkout response) to the browser.
* The browser returns this payload to the merchant (or PSP) via Payment Request.

#### Repeat purchase user journey

* The merchant (or PSP) creates a payment request with one or more SRC payment method identifiers. 
* The merchant (or PSP) calls hasEnrolledInstrument to see whether the user is ready to make a payment with SRC. The browser accesses the information it previously stored about the recognized user and returns "true".
* The user pushes the "buy" button, which calls show().
* The browser facilitates end to end UX while integrating with SRC System(s)
   * **Note**: UX includes card selection, possible challenge based on SRC System response, as well as confirmation UX for user to confirm (e.g. via local authentication)
* On user confirmation of SRC transaction, browser sends the request to SRC system. SRC system sends opaque payload (checkout response) to the browser.
* The browser returns this payload to the merchant (or PSP) via Payment Request.

 

#### Other Considerations

 

* What if browser does not implement this natively?  The fallback approach is as described in "Common Payment Handler" Approach.

* What if a browser only supports subset of card brands?

* What are the downsides of this approach?  This requires deeper browser integration with SRC Systems. 

* What are the possible limitations compared to existing SRC UX?  SRC UX supports "get list of instruments" equivalent API that merchant can invoke.  This continues to be missing, but if enabled, browser can natively return those instruments. 
# Obligations Implied by this Proposal

## Common Payment Handler Distributor

* Create and maintain a common origin.
* Maintaining a payment handler with the functionality described above.

## SRC system owners

* Take responsibility for the Payment Method Identifiers and Payment Method Manifests at the common origin.

## SRC-I Distributors (other than implementing SRC functionality)

* Supporting requests from and responses to the Common Payment Handler
* Supporting events on shipping address/option selection

## Browser makers

* Just in time install (supported in Chromium). 
   * **Note** There is an expectation that the browser will show each payment handler once for just-in-time installation, even if that handler is referenced by more than one applicable payment method manifest.
* Skip the sheet (supported in Chromium). 
* UNDER DISCUSSION: User-configurable "preferred payment handler". In an ecosystem with a Common payment handler, there may not be a need to implement a preferred payment handler.
* Optional: creating a built-in payment handler

## Registry Maintainer (Optional)

* Although it is not strictly speaking necessary, it may benefit the community if an entity such as EMVCo or W3C maintains a list of payment method identifiers for SRC systems.

# Discussion Topics

## Payer identity in Web-based payment handlers

We have discussed three requirements related to persistence of information, no exfiltration, and no user gesture upon return.

At the current time, we expect implementations to use indexDB for persistent storage (because cookies are not available to service workers). 

However, discussions are ongoing about other ideas (including Web Authentication).

# FAQ

We document here some general questions we have received about the proposal.

## If a payment handler supports N payment methods that are accepted by the merchant, does it receive the request data for all N?

A. Yes. Data associated with all of the matching payment methods will be sent to the payment handler in the methodData array of the PaymentRequestEvent.

## How can payment handlers (as service workers) persist information?

A. For the above proposal, we would expect that a payment handler from origin A would store information about the user, cards, authentication, etc. in IndexDB. 

Service workers do not have access to cookies, but they do have access to IndexDB (see [MDN documentation on IndexDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB)). All web content from the same origin can access the same IndexedDB. Therefore, if an origin registers multiple payment handlers (one per card), the data in IndexDB will need to be structured to enable each payment handler to retrieve the information relevant to it. For example, each payment handler could use the instrument ID as a key to separate the data in the IndexedDB, but there is no fundamental separation in storage (i.e. no privacy separation) between the two payment handlers from the same origin.

## Will this work if a user has more than one card?

A. Yes. The SRC-Is handle card enrollment and selection.

## How can user with two cards prevent a payment handler from associating them via these protocols?

A. Via a user profile (e.g., browser profile or OS profile).

## Can the payment handler notify the user of changes to an instrument?

A. Yes, via the [Push API](https://www.w3.org/TR/push-api/). All push notifications must be shown to the user and the user has to opt-in. The payment handler can customize the content of the notification. For more information, see [Web Push Notifications: Timely, Relevant, and Precise](https://developers.google.com/web/fundamentals/push-notifications).

## How does the SRC-I return data to the Common Payment Handler?

A. POST (over HTTPS) is one option. Additional security can be added using encryption in the application layer, namely symmetric key encryption between DCF and “Add Card” payment app using a shared secret key that was shared out of band. 

## Are payment handlers available in private browsing mode?

A. Yes. Note however, that:

* A payment handler will launch with no prior knowledge of instruments.
* Private browsing sessions are temporary. At the end of a private browsing session, information saved during the session will be discarded.

## When does Just-in-Time (JIT) Installation happen in Chrome?

A. JIT installation happens after the user selects the payment handler in the browser interface. If skip-the-sheet flow is triggered, then JIT installation happens upon show(). See, however, the [JIT proposal from March 2020](https://docs.google.com/document/d/1bzhh14E1DuJGYrueFhg87decGwvpPQz7D9mLzW8Yif4/edit?usp=sharing).

## What does hasEnrolledInstrument() return before the user has installed a payment handler?

A. If the payment handler is not yet installed, then hasEnrolledInstrument() will return "false". That's because hasEnrolledInstrument() relies on querying the payment handler through "canmakepayment" event, which is only possible after the payment handler has been installed. See, however, the [Payment Handler Availability Clarified Proposal](https://docs.google.com/document/d/1C_xH-6sJb9UedrvifsvqS_2k_We0IlPSfWFo8Kg54ps/edit) from March 2020.

## Can a payment handler store information and use it across different merchant origins?

A. Chrome's current implementation supports this, but is moving away from this. See the [storage proposal](https://github.com/w3c/payment-handler/wiki/2020-3p-context). Some browsers have begun to implement the [Storage Access API](https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API) which would give payment handler  the ability to access persistent storage across merchant origins. The user experience associated with Storage Access API may vary according to context (e.g., payments v. other types of activities) and is an active area of discussion.

## Will merchants be able to pass custom data to an SRC-I outside of the standardized EMVCo data set?

A. We expect the SRC payment method definition to support that. See [issue 26](https://github.com/w3c/src/issues/26).

## Who is the token requestor in this model?

A. Most likely the SRC system (per discussion at the [15 April 2020 task force meeting](https://www.w3.org/2020/04/15-wpwg-minutes.html).

## Could there be other payment handlers than those described?

A. This proposal sets an expectation that a payment handler is either:

* One authorized via payment manifests that live on a shared domain, or
* Built into the browser

That means this proposal does not anticipate other payment handlers.

However, nothing in the Web payments architecture prevents other parties from using Payment Request in other ways to do SRC. This would involve minting different payment method identifiers for use by merchants, and distributing other payment handlers. How entities are authorized to interact with SRC systems is outside of the scope of this document (and W3C work generally).