# Technical Design Template

## HTML Structure

```html
<h1>{{TITLE}} Technical Design</h1>
<p><strong>Author:</strong> Connor Marchand</p>
<p><strong>Date:</strong> {{DATE}}</p>
<p><strong>Status:</strong> Draft</p>
<p><strong>Related Documents:</strong> <a href="{{DISCOVERY_LINK}}">{{TITLE}} Technical Discovery</a></p>
<hr>
<h2>Overview</h2>
<p>{{2-3 sentence summary of what is being built and why}}</p>
<hr>
<h2>Background</h2>
<p>{{Why this work is needed. Reference the discovery doc findings. Keep brief — discovery doc has the details.}}</p>
<hr>
<h2>Goals &amp; Non-Goals</h2>
<h3>Goals</h3>
<ul>
  <li>{{Specific, measurable outcome}}</li>
</ul>
<h3>Non-Goals</h3>
<ul>
  <li>{{What is explicitly out of scope for this design}}</li>
</ul>
<hr>
<h2>Proposed Design</h2>
<h3>High-Level Approach</h3>
<p>{{Narrative description of the solution. How does it work end-to-end?}}</p>
<h3>Service Changes</h3>
<!-- One subsection per affected service -->
<h4>{{service-name}}</h4>
<p>{{What changes in this service and why}}</p>
<h3>API / Proto Changes</h3>
<!-- gRPC proto definitions, REST endpoint specs -->
<table>
  <thead><tr><th>Field</th><th>Type</th><th>Required</th><th>Description</th></tr></thead>
  <tbody>
    <tr><td>field_name</td><td>string</td><td>yes</td><td>description</td></tr>
  </tbody>
</table>
<h3>Kafka / Event Changes</h3>
<p>{{New topics, changed schemas, producer/consumer changes}}</p>
<h3>Database / Schema Changes</h3>
<p>{{New tables, columns, indexes, migrations}}</p>
<hr>
<h2>Implementation Plan</h2>
<h3>Phase 1: {{Name}}</h3>
<ul>
  <li>{{Concrete step}}</li>
</ul>
<h3>Phase 2: {{Name}}</h3>
<ul>
  <li>{{Concrete step}}</li>
</ul>
<hr>
<h2>Testing Strategy</h2>
<ul>
  <li>{{Unit tests: what to cover}}</li>
  <li>{{Integration tests: what scenarios}}</li>
  <li>{{Manual verification steps}}</li>
</ul>
<hr>
<h2>Rollout &amp; Deployment</h2>
<p>{{Feature flags, gradual rollout, order of service deployments, rollback plan}}</p>
<hr>
<h2>Risks &amp; Mitigations</h2>
<table>
  <thead><tr><th>Risk</th><th>Severity</th><th>Mitigation</th></tr></thead>
  <tbody>
    <tr><td>{{risk}}</td><td>H / M / L</td><td>{{mitigation}}</td></tr>
  </tbody>
</table>
<hr>
<h2>Open Questions</h2>
<ul>
  <li>{{Unresolved decisions that need input}}</li>
</ul>
```

## Section Notes

- **Goals & Non-Goals**: Be specific. Non-goals prevent scope creep.
- **Service Changes**: One `<h4>` per affected service. Link to relevant code.
- **API/Proto Changes**: Include full field tables with types and required flags — this is the contract.
- **Implementation Plan**: Phases should be independently deployable where possible.
- **Rollout**: Always address order of service deployments and rollback.
- GitHub links format: `<a href="https://github.com/Stodge-Inc/{repo}/blob/main/{path}#L{line}">description</a>`
