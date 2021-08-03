# Vac RFCs

This repository contains specifications for the [Vac](https://vac.dev) project.
Vac is an R&D org creating modular p2p protocols for private, secure, and censorship-resistant communication.
A detailed, albeit slightly outdated (2019), explanation of Vac and its design goals can be found [here](https://vac.dev/vac-overview).

Vac RFCs ([Request for Comments](https://en.wikipedia.org/wiki/Request_for_Comments)) includes specs for the [Waku family of specs](https://rfc.vac.dev/spec/10/),
but also other things such as:
node discovery, data sync, recommendations around usage, spec process, interfacing with auxiliary systems such as distributed storage, payload encryption, and so on.

Vac, while having a core team of maintainers and contributors, is an open and permission-less organization.

**See [rfc.vac.dev](https://rfc.vac.dev) for an easy to browse index of all RFCs.**

## Contributing

Please see [1/COSS](https://rfc.vac.dev/spec/1/) for general guidelines and spec lifecycle.

Feel free to join the [Vac discord](https://discord.gg/Vy54fEWuqC). There's a channel specifically for RFC discussions.

Here's the project board used by core contributors and maintainers: https://github.com/orgs/vacp2p/projects/5

### Building locally

Ensure you have the Hugo extended edition
(https://gohugo.io/getting-started/installing/), then run `hugo server`.

These protocols define various components of the [vac](https://vac.dev) stack.

### Style guide

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`.
Both the source and generated image should be in source control.
For ease of readability, the generated image is embedded inside the main spec document. 

Alternatively, [mscgenjs](https://github.com/mscgenjs/mscgenjs-cli) can be used to generate sequence diagrams (mscgenjs produces better quality figures especially concerning lines' spaces and figures' margins).
Once installed, the following command can be used to generate the sequence diagrams `mscgenjs -T png -i input.msc -o output.png`.
More details on the installation and compilation are given in [mscgenjs repository](https://github.com/mscgenjs/mscgenjs-cli).
You may try the online playground https://mscgen.js.org/ as well to get a sense of the output figures. 

## Acknowledgement

Site layout and process inspired by https://rfc.zeromq.org/
