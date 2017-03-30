---
layout: post
title: "Live updates to Wikimedia projects with EventStreams"
date: 2017-04-01 11:43
comments: true
categories: kafka wikimedia recentchanges sse eventsource event stream
---

_This was originally published on [Wikimedia's blog](https://blog.wikimedia.org/2017/03/20/eventstreams/)_

_Wikimedia's new public service that exposes live streams of Wikimedia projects is already powering several visualizations, like DataWaltz._

![Photo by Mikey Tnasuttimonkol, CC BY-SA 4.0.](https://wikimediablog.files.wordpress.com/2017/03/data_waltz_exhibit_in_la_2.jpg?w=580&h=386)

We are happy to announce [EventStreams](https://wikitech.wikimedia.org/wiki/EventStreams), a new public service that exposes live streams of Wikimedia events.  And we don’t mean the next big calendar event like the Winter Olympics or Wikimania.  Here, an ‘event’ is defined to be a small piece of data usually representing a state change. An edit of a Wikipedia page that adds some new information is an ‘event’, and could be described like the following:
```
{
    "event-type": "edit",
    "page": "Special Olympics",
    "project": "English Wikipedia",
    "time": "2017-03-07 09:31",
    "user": "TheBestEditor"
}
```

This means: “a user named ‘TheBestEditor’ added some content to the English Wikipedia’s [Special Olympics](https://en.wikipedia.org/wiki/Special_Olympics) page on March 7, 2017 at 9:31am”.
While composing this blog post, we sought visualizations that use EventStreams, and found some awesome examples.

[Open now](https://woodbury.edu/event/data-waltz-wuho/) in Los Angeles, [DataWaltz](http://datawaltz.hatnote.com/) is a physical installation that “creates a spatial feedback system for engaging with Wikipedia live updates, allowing visitors to follow and produce content from their interactions with the gallery’s physical environment.” You can see a photo of it at the top, and a 360 video of it [over on Vimeo](https://vimeo.com/208084520).

[Sacha Saint-Leger](https://github.com/sachaysl) sent us this [display of real-time edits on a rotating globe](https://sachaysl.github.io/wikimedia-challenge/), showing off where they are made.

!(https://wikimediablog.files.wordpress.com/2017/03/eventstreamsglobe.gif?w=960&h=1030)

[Ethan Jewett](http://github.com/esjewett) created a really nice [continuously updating chart](https://esjewett.github.io/wm-eventsource-demo/) of edit statistics.
!(https://wikimediablog.files.wordpress.com/2017/03/eventstreamseditcharts.gif?w=960&h=544)

## A little background—why EventStreams?

EventStreams is not the first service from Wikimedia to expose [RecentChange](https://www.mediawiki.org/wiki/Manual:Recent_changes) events as a stream. [irc.wikimedia.org](https://wikitech.wikimedia.org/wiki/Irc.wikimedia.org) and [RCStream](https://wikitech.wikimedia.org/wiki/RCStream) have existed for years.  These all serve the same data: RecentChange events.  So why add a _third_ stream service?

Both irc.wikimedia.org and RCStream suffer from similar design flaws.  Neither service can be restarted without interrupting client subscriptions.  This makes it difficult to build comprehensive tools that might not want to miss an event, and hard for WMF engineers to maintain. They are not easy to use, as services require several programming setup steps just to start subscribing to the stream.  Perhaps more importantly, these services are RecentChanges specific, meaning that they are not able to serve different types of events. EventStreams addresses all of these issues.

EventStreams is built on the w3c standard [Server Sent Events (SSE)](https://en.wikipedia.org/wiki/Server-sent_events).  SSE is simply a streaming HTTP connection with event data in a particular text format.  Client libraries, usually called EventSource, assist with building responsive tools, but because SSE is really just HTTP, you can use any HTTP client ([even curl](https://wikitech.wikimedia.org/wiki/EventStreams#Command-line)!) to consume it.

The SSE standard defines a Last-Event-ID HTTP header, which allows clients to tell servers about the last event that they’ve consumed.  EventStreams uses this header to begin streaming to a client from a point in the past.  If EventSource clients are disconnected from servers (due to network issues or EventStreams service restarts), they will send this header to the server and automatically reconnect and begin from where they left off.

EventStreams can be used to expose any useful streams of events, not just RecentChanges.  If there’s a stream you’d like to have, [we want to know about it](https://www.mediawiki.org/wiki/Topic:Tkjk4ezxb4u01a61).  For example, soon [ORES revision score events](https://ores.wikimedia.org/) may be exposed in their own stream.  The [service API docs](http://stream.wikimedia.org/?doc) have an up to date list of the (currently limited) available stream endpoints.

We’d like all RecentChange stream clients to switch to EventStreams, but we recognize that there are valuable bots out there running on irc.wikimedia.org that we might not be able to find the maintainers of.  We commit to supporting irc.wikimedia.org for the foreseeable future.
However, we believe the list of (really important) RCStream clients is small enough that we can convince or help folks switch to EventStreams.  We’ve chosen an official RCStream decommission date of July 7 this year.  If you run an RCStream client and are reading this and want help migrating, please [reach out to us](https://www.mediawiki.org/wiki/Topic:Tkjkee2j684hkwc9)!

## Quickstart

EventStreams is really easy to use, as shown by this quickstart example in JavaScript.  Navigate to [http://wikimedia.org/](http://wikimedia.org) in your browser and open the development console (for Google Chrome: More Tools > Developer Tools, and click ‘console’ on the bottom screen, which should open on the browser below the page you are visiting). Then paste the following:

```javascript
// This is the EventStreams RecentChange stream endpoint
var url = 'https://stream.wikimedia.org/v2/stream/recentchange';
// Use EventSource (available in most browsers, or as an
// npm module: https://www.npmjs.com/package/eventsource)
// to subscribe to the stream.
var recentChangeStream = new EventSource(url);
// Print each event to the console
recentChangeStream.onmessage = function(message) {
//Parse the message.data string as JSON.
var event = JSON.parse(message.data);
console.log(event);
};
```

You should see RecentChange events fly by in your console.

That’s it!   The [EventStreams documentation](https://wikitech.wikimedia.org/wiki/EventStreams) has in depth information and usage examples in other languages.

If you build something, please tell us, or add yourself to the [Powered By EventStreams wiki page](https://wikitech.wikimedia.org/wiki/EventStreams/Powered_By).  There are already some amazing uses there!
