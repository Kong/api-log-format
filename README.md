# API Log Format (ALF)

The API Log Format (**ALF**) is a new HTTP logging format that is loosely based on [HTTP Archive Format (HAR)][har-spec] with slight modifications for simplicity.

## Versions

- [v2.0.0](versions/2.0.0.md) - *Proposed*
- [v1.1.0](versions/1.1.0.md) - **Current**
- [v1.0.0](versions/1.0.0.md) - *Deprecated*
- [v0.0.1](versions/0.0.1.md) - *Deprecated*

## Tools

### Validation

The official validator is available at [Mashape/alf-validator][alf-validator]

### Analytics

Mashape's [Galileo][galileo] is the Analytics Platform for Monitoring, Visualizing and Inspecting API & Microservice Traffic, utilizes the ALF Format for reporting HTTP traffic using a variety of [agents][agent-spec].

[alf-validator]: https://github.com/Mashape/alf-validator "Official Validator"
[agent-spec]: https://github.com/Mashape/galileo-agent-spec
[har-spec]: https://github.com/ahmadnassri/har-spec "Har Specification"
[galileo]: https://getgalileo.io/ "Galileo"
