# QBit's Philosophy and inspiration

QBit philosophy:
====
At the end of the day QBit is a simple library not a framework.
Your app is not a QBit app but a Java app that uses the QBit lib.
QBit allows you to work with Java UTIL concurrent, and does not endeavor to hide it from you.
Just trying to take the sting out of it.

Inspiration
====

A big inspiration for Boon/QBit was Akka, Go Channels, Active Objects, Apartment Model Threading, Actor,
and the Mechnical Sympathy papers.

"I have read the AKKA in Action Book. It was inspiring, but not the only inspiration for QBit.".
"I have written apps where I promised a lot of performance and the techniques from QBit is how I got it."
 - Rick Hightower

QBit has ideas that are similar to many frameworks. We are all reading the same papers.
QBit got inspiration from the LMAX disruptor papers and this blog post about [link transfer queue versus disruptor](http://php.sabscape.com/blog/?p=557). We had some theories about queues that blog post insprired us to try them out. Some of these theories are deployed at some of the biggest middleware backends and whose name brands are known around the world. And thus QBit was born.

QBit also took an lot of inspiration by the great work done
by Tim Fox on Vertx. The first project using something that could actually be called QBit (albeit early QBit) was using Vertx on an web/mobile microserivce for an app that could potentially have 80 million users.
It was this experience with Vertx and early QBit that led to QBit development and evolution. QBit is built on the
shoulders of giants.
