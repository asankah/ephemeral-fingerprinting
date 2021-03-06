Ephemeral Fingerprinting On The Web
===================================

TL;DR:

-   Any web observable property whose changes are concurrently
    observable by multiple top-level sites can lead to cross site
    identity joining.

-   This method of identity joining does not require additional
    coordination between multiple first parties. A single third party
    embedded within multiple first parties can also use this method.

### Background

![diagram of site isolation boundaries with
overlap](/images/isolation-boundaries.png)

**Figure 1**: Two sites observe a sequence of device orientation changes
at times 𝒕₀, 𝒕₁,𝒕₂ .

All sites on the same UA instance share a clock and therefore can agree
on the timestamps with a small margin of error. The triplet 𝒕₀, 𝒕₁,𝒕₂
has a high probability of uniquely identifying the user. The two sites
can thus use these observations to conclude that the observations
originate from the same user.

As illustrated above, one or more low entropy signals observed
concurrently can be used to identify a user with a high degree of
confidence. Let’s call these **ephemeral fingerprints**. This document
discusses two types:

1.  *The sequence of timestamps corresponding to observed changes of a
    volatile surface can be used for identification*. Let’s call these
    **correlated events**.

2.  *A stream of observations of a volatile surface can be identifying*.
    Let’s call these **unique event streams**.

Signals considered for ephemeral fingerprinting don’t need to be highly
identifying by themselves. The [privacy budget
proposal](https://github.com/bslassey/privacy-budget) does not
adequately account for fingerprinting based on concurrent observations
of low entropy signals.

Device orientation, from our earlier example, can take one of two values
(portrait or landscape) and is unstable. Thus a single sample of device
orientation carries almost no information. I.e. A recorded observation
of device orientation doesn’t help at all with identifying the user at a
later time. However the timestamps corresponding to orientation changes
could have identifying levels of entropy.

These are not new. For example, this is discussed by Van Goethem et.
al. [1]

who calls these “Cross-Session Events” (§ 5 of linked paper). Potential
ephemeral fingerprinting surfaces also get flagged during
standardization discussions ( [Example: Polling
enumerateDevices](https://github.com/w3c/mediacapture-main/issues/403),
[Example: Ambient light
events](https://lists.w3.org/Archives/Public/public-privacy/2013JanMar/0007.html)).

#### Modelling Correlated Events

A correlatable event can be thought of as the tuple
`<surface-sample, timestamp>`. The addition of the timestamp strictly
increases the amount of information carried by the surface sample.

#### Modelling Event Streams

An event stream is simply a list of observed samples
`sample₀, sample₁, ...`.

Each additional observation strictly increases the amount of
information.

#### Other Examples:

-   `Accelerometer` properties.
-   `Sensor`, `onreading` event.
-   `BatteryManager.onlevelchange` : Deprecated but still shipping.
-   `Bluetooth.onadvertisementreceived`
-   `BroadcastChannel`, all events.
-   `MediaDevices.devicechange` event.
-   `GlobalEventHandlers.onfocus` and `onblur` events can fire
    simultaneously when switching between two browser windows.

### Mitigation

#### Permissions

**Goal:** Require informed consent from users.

There’s precedent for considering permissions[2] to be sufficient
mitigation for similar issues. For example, the Media Capture API
specification includes the following:

> For origins to which permission has been granted, the devicechange
> event will be emitted across browsing contexts and origins each time a
> new media device is added or removed; user agents can mitigate the
> risk of correlation of browsing activity across origins by fuzzing the
> timing of these events.

<p style="text-align: right">
From §15 of [Media Capture and Streams
API](https://w3c.github.io/mediacapture-main/#privacy-and-security-considerations)
specification.
</p>

**Pros**

-   Prevents drive-by fingerprinting.

**Cons**

-   UI for permissions don’t disclose the fact that all sites that have
    been granted access to the same resource can synchronize identifiers
    as a side-effect.

-   Doesn’t prevent identity joining once permission is granted.

#### Fuzzing Timing of Events

**Goal**: Deter correlation of events by injecting timing skew.

Mentioned in the snippet above from the Media Capture and Streams API
and called out by Jeffrey Yasskin as a potential general mitigation in
“desynchronize whole-browser events” in [this
issue](https://github.com/whatwg/html/issues/5215) filed against the
WHATWG HTML specification.

**Pros**

-   Lowers confidence of identity equivalence.

**Cons**

-   Precise timing may not be required. I.e. the lowered level of
    confidence could still be sufficient for most uses.

-   Doesn’t address unique event streams.

#### First-Party Restriction for APIs

**Goal**: Deter identity correlation by third-party sites.

Restrict APIs to the origin of the [top-level browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context).

The latter may choose to explicitly delegate access to the APIs via
[feature policies](https://w3c.github.io/webappsec-feature-policy/). But
third-party contexts can’t “reach across” browsing contexts via
correlation of cross context events or attributes that may be made
available by the API.

**Pros**

-   Makes it a requirement that first-parties actively cooperate with
    other first-parties or third-parties. I.e. Good first-party + bad
    third-party = safe.

**Cons**

-   Difficult to retrofit into existing APIs.

-   Browser-wide events and attributes are still visible to distinct
    top-level browsing contexts.

-   In practice top-level browsing contexts contain a fair amount of
    third party scripts which may access the same APIs. Furthermore
    there are financial incentives for first-parties to delegate API
    access to third-parties.

#### Limit API Access To Visible Browsing Contexts

**Goal:** Prevent background browsing contexts from skimming
identifiable events.

The Page Visibility API defines the [visibility state of a
document](https://w3c.github.io/page-visibility/#visibility-states) as
`visible` if the document is *“at least partially visible on at least
one screen”*.

Restrict APIs to — possibly top-level — browsing context’s active
document.

**Pros**

-   Is a convincing mitigation on mobile devices where only one
    top-level document can be visible at the same time.

**Cons**

-   There could be multiple visible browsing contexts which still leaves
    the door open for identity correlation across site boundaries.

-   The fact that having more than one browsing context open at the same
    time is a privacy risk is quite surprising for users.

#### Limit Events To Focused Top-Level Browsing Context

**Goal**: Limit firing correlatable events to a single [top-level
browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context).

The HTML spec defines a concept of a *[currently focused area of a
top-level browsing
context](https://html.spec.whatwg.org/multipage/interaction.html#currently-focused-area-of-a-top-level-browsing-context)*.
As defined, every top level [browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context)
has one regardless of visibility. A similar narrow concept could be
introduced that recognizes the top level browsing context that has
system input focus. There should be only one of these on a single
device.

*Let’s call the top-level browsing context that has system input focus
as the* **focused top-level browsing context**.

New specifications could restrict browser-wide events to the focused
top-level browsing context.

**Pros**

-   Likely fits in well with the intended usage model for most APIs.

**Cons**

-   Does not address event stream fingerprinting via polling.

#### Limit API Access To Focused Top-Level Browsing Context

**Goal**: Limit access to sensitive APIs to a single [top-level browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context).

Similar to the above, but addresses issues around polling by disallowing
access to the entire API or sensitive attributes by restricting the
entire API instead of just events.

**Pros**

-   Resilient to event stream fingerprinting via polling.

**Cons**

-   Can’t be easily retrofitted to existing APIs since it requires
    defining behavior for “disabled” APIs.

-   May break critical use cases.

#### Secure-Context Restriction and Control via Feature-Policy

These should be pretty standard at this point.

**Pros**

-   Just makes sense.

**Cons**

-   Not sufficient by itself.

### Spotting Ephemeral Fingerprinting Surfaces In Web Specs

**Ephemeral fingerprints:**

-   Require that multiple browsing contexts observe the same events or
    access the same volatile attribute.

-   These browsing contexts could involve a single third party in
    multiple first party contexts.

-   Does not require precise clocks nor agreement on the exact
    timestamps of the observed events. Depending on the fingerprintable
    surfaces involved the sequence of events could be identifiable by
    itself. Servers can roughly bucket observations by time periods,
    eliminating the need for client-side clocks.

-   Does not require an API to fire an event. A property with a volatile
    value that can be polled periodically is sufficient.

#### What to look for:

-   Events with external triggers. E.g. hardware based events.

-   Multiple distinct events that are fired in tandem to distinct
    browsing contexts. E.g. any processing model where multiple events
    are fired.

-   Volatile attributes that are visible across browsing contexts.

-   Volatile attributes that don’t share values across browsing
    contexts, but change simultaneously. E.g. Salting a volatile
    attribute is insufficient if its mutations can be correlated.

#### Example

Consider `onfocus` and `onblur` events.

The [focus update
steps](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps)
involve firing up to three distinct events: `change` if the node losing
focus is an `input` element, `focus`, and `blur`.

When focus traverses a browsing context boundary, these events may be
fired simultaneously to two different browsing contexts. Browsers
mitigate this by not firing `blur` for cross site tab switches, but they
still fire `blur` when the browser itself goes out of focus. Thus
identity can be correlated when switching browser windows.

##### Possible Mitigation

When the *new chain* and the *old chain*[3] are in different top-level
browsing contexts whose active documents are not same-origin, queue but
don’t fire `change` and `blur` events until focus returns to the old
top-level browsing context.

#### Notes

[1] Van Goethem, T. and Joosen, W., 2017. One side-channel to bring them
all and in the darkness bind them: Associating isolated browsing
sessions. In *11th {USENIX} Workshop on Offensive Technologies ({WOOT}
17)*.
([PDF](https://pdfs.semanticscholar.org/5814/9610a57cb4626918bf003b8bad25e740b1f4.pdf))

[2] Either via a legacy permissions prompt or explicitly requiring the
use of the [Permissions API](https://w3c.github.io/permissions/) in the
spec for sensitive APIs.

[3] *New chain* and *old chain* are defined in [focus update
steps](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps).
