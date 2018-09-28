# Kubernetes LAN Party

This repo contains all of the info and configuration needed to get your LAN party running on Kubernetes. Or, rather, it will soon. Work in progress!

## Who is this for

LAN parties have a few constraints that make Kubernetes not as straight forward as other use cases. For example, game servers need to be on the same broadcast domain to be discovered in the LAN tab of games; and for games that don't support that, we need sane DNS addresses for humans to use. And numbered servers. And simple cluster storage. So this guide is for people who want to run Kubernetes in an environment like that.

## Why Kubernetes for your LAN Party?

Buzz words.

Also - Centralised management. Infrastructure as Code. Super fast and reproducible setup.

Kubernetes adds complexity, so it's not for everyone. Before using it, make sure you have a backup plan, just in case :)

## What is in this

* Documentation
* Scripts
* [Example configuration files](configs/)

## Documentation Index

* [Host Installation](installation.md)
* [Host and pod network configuration](network.md)
* [DNS](DNS.md)
* [Internal Docker registry](registry.md)
* [Kubernetes Dashboard](dashboard.md)
* [Persistent internal storage](storage.md)

## Join the discord!

[Join the Open Source LAN Discord!](https://discord.gg/0149LEvYPSzmnItKb). Chat with us about what you're doing with Kubernetes :)

## TODO

This repo is still a WIP and hasn't been used in production yet. More content coming.

TODO:

* Scripts to automate most of the work (eg host deployment)
* Deploying game servers
* Deploying a LAN cache server
* Documenting caveats
* Testing in production
* Much else...
