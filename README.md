# rfc.vac.dev

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

See [rfc.vac.dev](rfc.vac.dev).

## Contributing

Please see [1/COSS](https://rfc.vac.dev/spec/1/) for general guidelines and spec lifecycle.

### Building locally

Ensure you have the Hugo extended edition
(https://gohugo.io/getting-started/installing/), then run `hugo server`.

These protocols define various components of the [vac](https://vac.dev) stack.

### Style guide

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`. Both the source and generated image should be in source control. For ease of readability, the generated image is embedded inside the main spec document. 

Alternatively, [mscgenjs](https://github.com/mscgenjs/mscgenjs-cli) can be used to generate sequence diagrams (mscgenjs produces better quality figures especially concerning lines' spaces and figures' margins). Once installed, the following command can be used to generate the sequence diagrams `mscgenjs -T png -i input.msc -o output.png`. More details on the installation and compilation are given in [mscgenjs repository](https://github.com/mscgenjs/mscgenjs-cli). You may try the online playground https://mscgen.js.org/ as well to get a sense of the output figures. 

### Meta

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Acknowledgement

Site layout and process inspired by https://rfc.zeromq.org/
