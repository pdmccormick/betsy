The Betsy LED display
=====================

## What is _Betsy_?

_Betsy_ is a development prototype LED display from
[Luminautics](http://luminautics.com/), creators of rugged digital signage
technology. Originally manufactured in 2011 and consisting of 54 square feet of
high output, full colour RGB modular tiles, the technology at the heart of this
display went on to see commercial adoption in large format outdoor displays.

In August 2016, Betsy was donated to the [HackLab.TO](https://hacklab.to/), a
community hackerspace in Toronto, Ontario, Canada.

## Interacting with the display

Betsy can be directly driven through a simple UDP protocol over IPv6,
documented in [PROTOCOL.md](PROTOCOL.md). The easiest way to get started is
with the packages written in
[Python](https://github.com/pdmccormick/python-betsy) and
[Go](https://github.com/pdmccormick/go-betsy).

## Who is behind this

 * [Peter McCormick](https://github.com/pdmccormick/) wrote the firmware and host packages, as well as arranged the donation to the [HackLab.TO](https://hacklab.to/)
 * [Luminautics](http://luminautics.com/) also includes: Graham Murdoch, Leo Mordoukhovski, Cheng Qian and [David Wolever](https://github.com/wolever)

## Links

 * [python-betsy](https://github.com/pdmccormick/python-betsy): Python package for interacting with and controlling Betsy
 * [go-betsy](https://github.com/pdmccormick/go-betsy): Golang package for interacting with and controlling Betsy
