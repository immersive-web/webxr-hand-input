# WebXR Hand Input

The [WebXR Hand Input Specification][this-spec] adds hand input support in WebXR.
Feature lead is Manish Goregaokar ([@Manishearth](https://github.com/Manishearth)). 

## Taking Part

1. Read the [code of conduct][CoC]
2. See if your issue is being discussed in the [issues][this-spec], or if your idea is being discussed in the [proposals repo][cgproposals].
3. We will be publishing the minutes from the bi-weekly calls.
4. You can also join the working group to participate in these discussions.

## Specifications

* [WebXR Hand Input][this-spec]: Hand input support in WebXR
* [Explainer](explainer.md)


### Related specifications
* [WebXR Device API - Level 1][webxrspec]: Main specification for JavaScript API for accessing VR and AR devices, including sensors and head-mounted displays.

See also [list of all specifications with detailed status in Working Group and Community Group](https://www.w3.org/immersive-web/list_spec.html). 

## Relevant Links

* [Immersive Web Community Group][webxrcg]
* [Immersive Web Working Group Charter][wgcharter]
* [Originating proposal](https://github.com/immersive-web/proposals/issues/48)

## Communication

* [Immersive Web Working Group][webxrwg]
* [Immersive Web Community Group][webxrcg]
* [GitHub issues list](https://github.com/immersive-web/layers/issues)
* [`public-immersive-web` mailing list][publiclist]

## Maintainers

To generate the spec document (`index.html`) from the `index.bs` [Bikeshed][bikeshed] document:

```sh
bikeshed spec
```

## Tests

For normative changes, a corresponding
[web-platform-tests][wpt] PR is highly appreciated. Typically,
both PRs will be merged at the same time. Note that a test change that contradicts the spec should
not be merged before the corresponding spec change. If testing is not practical, please explain why
and if appropriate [file a web-platform-tests issue][wptissue]
to follow up later. Add the `type:untestable` or `type:missing-coverage` label as appropriate.


## License

Per the [`LICENSE.md`](LICENSE.md) file:

> All documents in this Repository are licensed by contributors under the  [W3C Software and Document License](https://www.w3.org/Consortium/Legal/copyright-software).

# Summary

For more information about this proposal, please read the [explainer](explainer.md) and issues/PRs.

<!-- Links -->
[this-spec]: https://immersive-web.github.io/webxr-hand-input
[CoC]: https://immersive-web.github.io/homepage/code-of-conduct.html
[webxrwg]: https://w3.org/immersive-web
[cgproposals]: https://github.com/immersive-web/proposals
[webxrspec]: https://immersive-web.github.io/webxr/
[webxrcg]: https://www.w3.org/community/immersive-web/
[wgcharter]: https://www.w3.org/2020/05/immersive-Web-wg-charter.html
[webxrref]: https://immersive-web.github.io/webxr-reference/
[publiclist]: https://lists.w3.org/Archives/Public/public-immersive-web-wg/
[bikeshed]: https://github.com/tabatkins/bikeshed
[wpt]: https://github.com/web-platform-tests/wpt
[wptissue]: https://github.com/web-platform-tests/wpt/issues/new
