# Ad Topic Hints

A [Proposal](https://privacycg.github.io/charter.html#proposals)
of the [Privacy Community Group](https://privacycg.github.io/).

## Authors:

- [Benjamin Savage](https://github.com/benjaminsavage)
- [Andrew Knox](https://github.com/ajknox)
- [Sean Bedford](https://github.com/bedfordsean)
- [Erik Taubeneck](https://github.com/eriktaubeneck)

## Participate
- https://github.com/privacycg/ad-topic-hints/issues
## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [The API](#the-api)
  - [getAdTopicHint](#getadtopichint)
  - [addFeedback(domElementOfAd, typeOfFeedback)](#addfeedbackdomelementofad-typeoffeedback)
- [Additional functionality](#additional-functionality)
  - [Mitigating the risk of click-jacking](#mitigating-the-risk-of-click-jacking)
  - [When no buttons are present](#when-no-buttons-are-present)
  - [Export / Import](#export--import)
- [Implementation Details](#implementation-details)
  - [Privacy Constraints](#privacy-constraints)
  - [Binary embedding vectors](#binary-embedding-vectors)
- [Key scenarios](#key-scenarios)
  - [User wants to see more or less of a type of ad](#user-wants-to-see-more-or-less-of-a-type-of-ad)
  - [User wants to manually modify their ads preferences](#user-wants-to-manually-modify-their-ads-preferences)
- [Detailed design discussion](#detailed-design-discussion)
  - [How the ad topic embedding is trained](#how-the-ad-topic-embedding-is-trained)
  - [Updating the embedding](#updating-the-embedding)
  - [Representing ad topics in human language](#representing-ad-topics-in-human-language)
  - [Why am I seeing this?](#why-am-i-seeing-this)
  - [How are *random* ads selected?](#how-are-random-ads-selected)
  - [How could the user indicate a preference to "explore" the ads space outside of their current choices?](#how-could-the-user-indicate-a-preference-to-explore-the-ads-space-outside-of-their-current-choices)
  - [Should a user's feedback be permanent or time-limited?](#should-a-users-feedback-be-permanent-or-time-limited)
  - [Should ads preferences be scoped at an eTLD+1, user-agent level, device level, or user level?](#should-ads-preferences-be-scoped-at-an-etld1-user-agent-level-device-level-or-user-level)
- [Considered alternatives](#considered-alternatives)
  - [Is there a problem with advertisers and publishers learning about which people provided feedback on their ads?](#is-there-a-problem-with-advertisers-and-publishers-learning-about-which-people-provided-feedback-on-their-ads)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction
One of the most important advertising use-cases is to select ads that are relevant to the viewer. Irrelevant ads are an annoyance for people using the web, a waste of ad budget for the advertisers, and a missed opportunity for publishers. There are a variety of approaches to try to select and show more relevant ads:

1. **Behavioral advertising** utilizes data about a person's behaviors to select ads. 
2. **Contextual advertising** utilizes the context of where an ad is shown to select ads. 
3. We'd like to introduce the term **"Person-driven advertising"** to describe an ad selection system which utilizes a person's explicitly stated feedback about what ads they would like to see to select ads.

As browsers make changes to prevent cross-site tracking, behavioral advertising will become more difficult. "Person-driven advertising" seems like an interesting alternative worth exploring. 

This proposal aims to give web users a mechanism by which they can provide feedback on the relevance of the topics of the ads they see, and a mechanism by which their user-agent can communicate their preferences to websites to assist with the selection of relevant ads.
## Goals

* Give people the power to influence the topics of ads they see across the web
* Help ad-funded websites select relevant ads and thereby monetize more effectively
* Preserve user privacy, specifically prevent cross-site tracking of users.

## Non-goals

* Dictate any sort of policy for how websites must select ads
* Assist websites with fulfilling legal obligations (e.g. Do not show ads for alcoholic beverages to minors)

## The API

### getAdTopicHint

```js
const ad_topic_hint = window.getAdTopicHint();

// custom logic to utilize this ad topic hint in ad selection
const ads = await makeAdRequest(contextual_data, ad_topic_hint);

```
The `window.getAdTopicHint()` returns an ArrayBuffer. This should be thought of as a "binary embedding vector" of length N. Another way of saying this, is that it is a vector in an N dimensional space, where each dimension is constrained to either the value 1 or 0. N is probably in the range of 64 to 1024. See the "Binary embedding vectors" section below for a discussion on this topic. See the "Privacy Constraints" section below on how this vector is selected.
### addFeedback(domElementOfAd, typeOfFeedback)

The key design consideration is the mechanism by which people choose which ad topics they want to see. We propose that the mechanism by which this is accomplished is through feedback about specific ads. 

The user experience should be designed to minimize friction in providing feedback. As such, inline buttons which only require one click/tap seem like a better approach than (for example) needing to navigate menus with smaller, or harder to tap entry points (e.g. the "Ad Choices" icon).

As such, the recommendation is for ads to have two highly visible buttons, one that people click to indicate that an ad is about a topic that is relevant to them, and another to indicate that an ad is about a topic which is not relevant to them.

To allow site owners to customize the ad experience of their website, the recommendation is for websites to create their own buttons which call a modern Javascript API to specify the DOM element containing the ad and type of feedback (positive or negative). 

```js
<script>
function provideFeedback(feedback_type) {
  const ad = document.getElementByID('ad12345');
  window.AdTopicHints.addFeedback(ad, feedback_type)"/>
}
</script>
<div id="ad12345>
  <a attributionsourceid="17" attributiondestination="https://destination.example/">
    <!-- any clickable parts of the ad go here, possibly including images / video --/>
  </a>
  <div id="ad12345_feedback">
    <input type="button" onClick="provideFeedback(window.AdTopicHints.POSITIVE_FEEDBACK)"/>
    <input type="button" onClick="provideFeedback(window.AdTopicHints.NEGATIVE_FEEDBACK)"/>
  </div>
</div>
```
When a person clicks / taps on either of these buttons, the browser should extract the innerHTML of the DOM element provided to the addFeedback() method. The intention is to capture the ad, as it looks to the end-user. This will be important for a few reasons that will be discussed below - including the need to show this ad to the user at a later time in a way that they will recognize it.

As mentioned above, the ad content should also be saved somehow, so that it can be shown again to the user in various contexts (e.g. if they ever choose to review previously provided ad feedback). There are tradeoffs between storing this remotely vs locally that need to be discussed.

One possible illustration of how custom-styled ad feedback buttons could potentially look.

![Mockup of an ad with “More of this” and “Less of this” buttons](https://github.com/privacycg/ad-topic-hints/blob/main/Jaspers%20Market.png)

## Additional functionality
### Mitigating the risk of click-jacking
Allowing for custom styling of the buttons opens up the potential for "click jacking" and other types of misleading experiences. Examples include:

* An invisible "ad feedback button" overlaid with absolute position, above other content the user is trying to click on.
* Styling the ad feedback buttons with text like "Hide Ad".
* Creating an "ad feedback button" which is adjacent to ad "A", but actually refers to ad "B"
* Creating an "adversarial ad" which visually appears to be about one topic, but which tricks the embedding system into thinking it is about a different topic.

To mitigate such risks, it seems necessary to have some type of visual feedback to the end user about the feedback they have just provided, with a confirmation that this is something they want to do.

This is in tension with the design goal of minimizing friction, and needs to be considered carefully to balance both goals. The implementation details are ultimately up to the browser, but a few recommendations:

1. A system dialog should be shown each time feedback is provided on an ad.
2. This dialog should show (visually) the ad about which feedback was just provided.
3. The dialog should indicate clearly if the feedback was "Relevant Topic" or "Irrelevant Topic"
4. The dialog should have some mechanism by which the user can confirm "Yes, that's right" or "No, I did not intend to provide this feedback".
5. The dialogue should also display a human readable topic string which can be communicated to the end-user. This should help with the risk of "adversarial ML".
6. **\[Optional\]** To minimize friction, perhaps this confirmation dialog could auto-close after a few seconds of no user interaction, like the "Screenshot" UX on iOS. 
7. **\[Optional\]** It might also be a good idea to include a "Manage prior feedback" link the user could click to open the appropriate menu in their browser that lists all saved ad topic feedback.

Here is a wireframe to stimulate discussion:

![Wireframe of confirmation dialogue](https://github.com/privacycg/ad-topic-hints/blob/main/Confirmation%20Dialogue.png)

### When no buttons are present
Allowing custom feedback buttons also means that developers might choose to not add any feedback buttons at all. 

Firstly, it is worth noting that there is no financial incentive on the publisher or ad network's part to prevent people from providing feedback. When people provide negative feedback on the topic of an ad, stating that it is "not relevant" to them, this does not limit the set of data which can be used in the future to select ads. If anything, it may improve ad selection in the future, by helping to avoid topics that are "not relevant" to the viewer. As such, we do not see any structural reason why publishers and ad networks would be averse to including feedback buttons.

That said, for whatever reason, publishers and ad networks may choose not to include ad feedback buttons. In this case, we recommend the browser provide an alternative mechanism by which people can provide feedback. While it may not be as frictionless as a single button click, it seems important for people to have at least some mechanism by which they can provide feedback on any ad they see on the internet. 

The exact user-experience should be left to the browser developer to design, but possible ideas include right-clicking on content and selecting options from a menu, or long-pressing on mobile. 
### Export / Import

People should be able to import or export their existing ad topic preferences. This will enable a number of workflows, such as:

* The ability to easily port their choices between platforms and ecosystems without having to start again. 
* The ability to share and download pre-customized "Ad Topic Profiles" that match their interests.
* Debugging or stress testing ad delivery systems.
* Research on the types of ads being delivered on the internet.

To enable this, we need a standardized file format to capture ad topic feedback. One way to do this would be to use JSON.

Minimally, this would require the ad content to be loaded by the browser, so that embeddings can be computed for those ads. Depending upon the previous design choice (store ads locally or remotely) this file might either need to contain serialized ads data or else permalinks.

The risk of saving the embedding in the file is that it could either be out of date (if the version of this API has been updated) or a malicious entity could intentionally provide embeddings which do not match the ads the user sees.

```js
{
    "version": "1.0",
    "more_like_ad_topics": [
        "ad_content": "...",
        "ad_content": "...",
        // etc...
    ],
    "less_like_ad_topics": [
        "ad_content": "...",
        "ad_content": "...",
        // etc...
    ]
}
``` 

## Implementation Details
### Privacy Constraints
Ad topics have quite a bit of entropy, so we need to be confident that releasing a topic hint from the API does not significantly enable new tracking capabilities. In order to accomplish this, we propose the following preliminary mechanism based on randomized response:

1. When a session starts, the browser generates several arbitrary and random prospective embedding vectors. The more dimensions we use in the embedding, the more random samples would need to be generated.
2. Each prospective hint is scored against the previous ad feedback (both positive and negative) that the user has provided. In this way, we can generate a "raw affinity" score per prospective hint.
3. The raw affinities are normalized into probabilities using the following relationships: All normalized affinities are positive, the sum of normalized affinities is 1, and the largest normalized affinity is at most e^ε larger than the smallest normalized affinity.
4. A topic hint is chosen from the prospective hints randomly, with a probability of the normalized affinity for that hint.
5. The browser uses that topic hint for the duration of the session.

This "sample and score" method allows us to generate topic hints from a very large embedding space, but still maintain an easily calculated ε differential privacy guarantee for any given session. This differential privacy guarantee provides a limit on how useful an ad topic hint would be for tracking a particular user across multiple websites. 

This is a basic approach we are suggesting just to start the conversion. We encourage more research and development into alternate methods of ad topic hint generation that may provide better utility/accuracy or less computational cost at comparable levels of privacy.
### Binary embedding vectors
In general, it is possible to compress any embedding down from a floating point representation to a binary representation. If using 4-byte floats, this equates to a 32x reduction in file-size, and often with only a minimal loss of quality. As representing an ad topic with an embedding vector may require anywhere from 64 to 1024 dimensions, this can be quite material.

Computational resources are also saved, as the "distance" between two binary vectors can be easily computed by use of the "Hamming Distance", which just counts the number of bit positions in which the two bit-strings are different.
## Key scenarios
### User wants to see more or less of a type of ad

As a user browses the internet, they will see ads. They should be able to tell the user-agent about the ad through an easy feedback mechanism and indicate that the topic of the ad is either "Relevant" or "Not relevant" to their interests. This feedback should be provided through the two buttons we previously discussed.

### User wants to manually modify their ads preferences
We envision a browser-provided UI which shows them the set of ads upon which they have previously provided feedback, along with the ability to modify this. We also envision the ability to import / export ad preferences. Since the representation of the users' preferences is entirely within their control in their browser, this allows people to set any set of ad preferences they desire. This is an intentional design decision. 
## Detailed design discussion
### How the ad topic embedding is trained
The most important characteristic of the embedding, is that _**ads that people perceived to be about the "same topic" have similar embeddings**_, and that _**ads that people perceived to be about "different topics" have dis-similar embeddings**_. It is not important to define the concept of a "topic" or to have any sort of topic hierarchy. Even the words to describe what topic the ad is about are not important for the purposes of training this embedding.

As such, all that is needed for "training data" is to collect feedback from representative web users about which ads they deem to be "about the same topic". We need to collect a long list of pairs of ads, where each pair are different ads that people perceive to be "about the same topic". We also need a list of ads "about different topics". We can either collect that as well, or select random pairs of ads from the previous list under the assumption that random ads are unlikely to be "about the same topic".

It is important to include training data from speakers of many languages and cultures, on ads for a wide variety of topics. Too small of a data set will lead to a lower quality embedding being trained.

Once we have this training data, we can train a neural network to minimize a cost function which is computed across the set of training data to minimize the "distance" between the embedding vectors of those ads deemed to be "about the same topic", and maximize the "distance" between the embedding vectors of ads deemed to be "about different topics".

| Ad A | Ad B | True label          | Predicted label | Error |
|------|------|---------------------|-----------------|-------|
| ad1  | ad2  | 1 (same topic)      | 0.9             | 0.1   |
| ad3  | ad4  | 0 (different topic) | 0.1             | 0.1   |
| ad5  | ad6  | 1 (same topic)      | 0.8             | 0.2   |
| ad7  | ad8  | 0 (different topic) | 0.2             | 0.2   |
| ad9  | ad10 | 1 (same topic)      | 0.85            | 0.15  |
|      |      |                     | Total Error:    | 0.75  |

An alternative approach to generate a similar set of training data would be to just look at what ads people click, or provide positive feedback upon. One could generate positive examples from pairs of ads that the same person clicks / liked, and negative examples from pairs of ads the same person did NOT click / like. This might be an easier method of generating a very large data set, and would not place any sort of power in the hands of any central entity, but it might not exactly map to the intended concept (are these ads the same topic?) It might represent something more along the lines of "are these ads likely to be interesting to the same type of person?".

Whatever approach is used, we recommend making the set of training data publicly available, as well as making the final trained neural network publicly available. 
### Updating the embedding
Over time, we may find deficiencies in the embedding. Perhaps certain ad topics are not well represented, or we are able to obtain larger and better quality datasets to train upon. As such, it may be desirable to update the embedding used from time to time. As such, it seems desirable to have some kind of version string available - so that all participants in the ads ecosystem can update their systems accordingly. This is another reason why browsers need to maintain a reference to the original ad - so that new embedding vectors can be computed if the ad to embedding mapping is changed. Overall, updates will be expensive, and should not happen very often.
### Representing ad topics in human language
It may be important to explain to web users in human language what Ad Topic a particular ad seems to be about. In fact, we proposed earlier that the confirmation dialog shown to users after providing ad feedback display the ad topic to mitigate the risk of adversarial ads.

We believe that the module that converts ad topic embeddings into human readable strings should be treated as a fully separate component, which can be easily substituted / swapped / customized. For example, there should be localized modules for each language. Each browser vendor might choose to implement this module differently. Finally, people might want to install custom extensions to customize this experience.

It may be desirable to expose this module directly to web developers so that they can themselves convert an embedding vector into human readable text in the locale of the end user. We can imagine uses in experiences like: "Why am I seeing this ad?". A simple API implementation of this could look like:

```js
  window.AdTopicHints.getReferenceDescriptor(ad_element)"/> 
  // Returns e.g. "Travel"
```

To implement this module, all that is needed is a "Golden Set" of examples. Given an embedding vector, the browser simply needs to compute the "distance" to all of the "Golden Set" examples and determine which one is "closest". The "Topic" of that example can then be shown to the user.

The composition of this "Golden Set" is not important for the operation of the core algorithm, and can be swapped out for a different set with minimal impact. The only reason this is needed is for any UX which needs to explain what is going on to the end user (e.g. the confirmation dialogue).

One imagines the composition of this "Golden Set" may vary dramatically according to language and culture. One also imagines there might be disagreement about how to best name various "Ad Topics". One way of approaching this is to leverage standards bodies and industry trade groups - setting up working groups aimed at arriving upon an agreed upon "Golden Set" for each language. Another way would be to allow many different entities to develop their own "Golden Set" and then testing these out in user research, to see which most closely aligns with the perception of representative web users. We are not particularly opinionated about how this is done.
### Why am I seeing this?
In addition to when web developers want to display the "human language" representation, we also recommend that the browser make an affordance to the user whenever an Ad Topic Hint is requested. This could include a disclaimer that the website is under no obligation to follow that suggestion, but would provide a baseline level of explainability to why the user is seeing ads on any given page.

### How are *random* ads selected?
From a differential privacy perspective, a purely random ad is ideal, but this may present some challenges. Ideally a website would be unable to determine if a given ad hint is random or real, and a website over time may learn that certain values are more likely to be random than others. One suggestion from the discussion in the PrivacyCG was to have the ad topic hint reflect the "average user", potentially within a certain demographic or region.

It would be difficult for the browser to infer this locally, but it seems like this could be accomplished using an MPC to compute the necessary statistics for the browser to generate a pseudo-random value that fits this specified profile.

### How could the user indicate a preference to "explore" the ads space outside of their current choices?
The differential privacy mechanism described in the "Privacy Considerations" section will naturally explore topics for which the user has not yet provided feedback.

We believe this is probably sufficient, but browsers could also innovate other ways for users to see outside their current preferences. Examples include: occasionally sending completely random ad topic hints ('explore-exploit') or special behavior for private browsing modes.

### Should a user's feedback be permanent or time-limited?
As a user interacts with this system and provides feedback, we have a consideration around how long to consider that feedback to be valid for. Two options come to mind;

1. User preferences should be considered static and always respected
2. User preferences may change over time and this should be considered through some form of weighting or "time decay" within the system

Arguments can be made for both approaches, for example; user choice is foremost and those choices should be respected. Alternatively; if a user is moving house and interested in sofas, they may state a preference for that, but without manually changing that preference later they will continue to get (potentially irrelevant) furniture ads forever.

Another consideration here is the overall size of stored ad topic preferences - if this is allowed to grow exponentially, it may at some point become unmanageable within the user-agent.
### Should ads preferences be scoped at an eTLD+1, user-agent level, device level, or user level?
We suggest that this would be most effective if scoped at the device level, with the ability to coordinate their preferences for all their devices using a "sync" or manual import/export function. Given the prevalence of in-app advertising, it would be beneficial for users and advertisers if that ecosystem was also able to interact with the ad topics hints system. This does need to take into consideration user expectations, but we believe that scoping to the device as default behavior benefits a diverse ad-supported app ecosystem.

We have no concerns with browsers providing the technical capability to customize Ad Topic Hints at an eTLD+1 level, but we are skeptical that most web users will have the desire and inclination to manage preferences at that granular of a level.
## Considered alternatives
### Is there a problem with advertisers and publishers learning about which people provided feedback on their ads?
While on the one hand this proposal provides a ton of customizability, user provided feedback about ads is not kept secret from the website where the ad is displayed. This is similar to ad clicks - which are also not kept a secret from the website where the ad is shown. 

If ad feedback is known to websites (including if it was positive or negative), we cannot technically "purpose limit" it to only be used for selecting ads more to the user's liking (the original purpose for which the user provided this information). 

While we have proposed a technical mechanism to try to prevent this data from being used for cross-site tracking purposes, it's conceivable that it could be used for other purposes as well. 

If one wanted to avoid this possibility, alternative designs would likely involve rendering ads within fenced frames, and potentially on-device ad auctions. There are many downsides to this approach. It's an important discussion, but our belief is that these costs are not worth it - and we should seek mechanisms like those described in the "Privacy considerations" section to limit potential misuse.
## Stakeholder Feedback / Opposition
New proposal - no signal yet.

- Chrome : No signals
- Safari : No signals
- Firefox : No signals
- Edge : No signals
## References & acknowledgements

Several people have provided valuable feedback already in the [Privacy CG issue](https://github.com/privacycg/proposals/issues/26) filed on Jun 21, 2021. We're thankful for all that engagement.

In addition, we’d like to call out these proposals that helped shape our thinking:

- [FLoC](https://github.com/WICG/floc)
- [PARAKEET](https://github.com/WICG/privacy-preserving-ads/blob/main/Parakeet.md)
