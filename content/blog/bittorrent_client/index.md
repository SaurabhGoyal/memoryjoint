+++
title = "Writing a Bittorrent Client"
date = 2024-07-05
description = "Writing a Bittorrent client."

[taxonomies]
tags = ["torrent", "networking", "io"]

[extra]
toc = true
quick_navigation_buttons = true
+++

# Bittorrent
**What is it?**
- It is a protocol to exchange data chunks with other people on a network, generally internet.

**How is it different from simple HTTP?**
- HTTP requires client server model where you (client) download data from a data owner (server) that you know about.
- Bittorrent enables a lot of people in the internet to collaboratively make the file available to a lot of other people without one server being a bottleneck or owner. Essentially one file can have many servers instead of one.
- This results in higher availability and download speed of the file.

**How does it work?**
- This is the best part - this is one of the simplest protocols I have ever read. This is essentially a simple logical solution to how to achieve multiple people in a network requesting and serving a file.
- The whole specification is listed [here officially](http://bittorrent.org/beps/bep_0003.html) and [here unofficially (but more verbose)](https://wiki.theory.org/BitTorrentSpecification#interested:_.3Clen.3D0001.3E.3Cid.3D2.3E) but I would give a high level view here -
    - You are a peer, looking for a file.
    - You need a unique-id for the file and someone with whom you can ask about servers of that file.
    - These two pieces of information is available in Torrent file / magnet link. You parse that file / link to fetch above two, i.e. unique-id (info-hash) and someone to enquire sources (tracker).
    - You ask tracker to provide a list of sources (IP addresses of other people in the network) who can give file.
    - You ping these peers and say that you need data for this file, if peer has it, they ping bank, i.e. handshake completed.
    - You ask for file data in smaller chunks from peers.
    - Peers send thbe requested smaller chunks if they have.
    - Once all chunks have been downloaded, you combine them into the file. !!! You have the file now. !!!
    - All this meanwhile, peer keeps telling you, which specific chunks it has and whether it can currently serve you those or not.
    - And the most important part, YOU ARE ALSO A PEER, serving file chunks that you have to other people requesting them.

**Why build a client?**
Building a robust client requires dealing with following concepts in depth -
- Reading protocol specifications and following them - As simple as it sounds, most of my time during implementation was wasted because I did not read the specification thoroughly enough and missed sending the correct message to peer and having wrong expectations from the peer without a clear way to debug.
- TCP/UDP - It requires a lot of data transfer and packet exchanges which need to be raw TCP/UDP messages. For them to be correctly perceived by peer, yo