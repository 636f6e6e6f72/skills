# Technical Discovery Template

## HTML Structure

```html
<h1>{{TITLE}} Technical Discovery</h1>
<p><strong>Author:</strong> Connor Marchand</p>
<p><strong>Date:</strong> {{DATE}}</p>
<p><strong>Status:</strong> Draft</p>
<p><strong>Related Documents:</strong> {{RELATED}}</p>
<hr>
<h2>Background</h2>
<p>{{BACKGROUND}}</p>
<hr>
<h2>Purpose &amp; Scope</h2>
<p>{{PURPOSE}}</p>
<h3>In Scope</h3>
<ul>
  <li>{{What is explicitly included in this discovery}}</li>
</ul>
<h3>Out of Scope</h3>
<ul>
  <li>{{What is explicitly excluded, though may be relevant in the future}}</li>
</ul>
<hr>
<h2>Objectives &amp; Questions</h2>
<p>The technical design resulting from this discovery must answer the following questions:</p>
<ul>
  <li>Question one? ✅</li>
  <li>Question two? 💬</li>
</ul>
<hr>
<h2>Activities &amp; Approach</h2>
<p>{{How research was conducted — code review, team meetings, POCs, etc.}}</p>
<hr>
<h2>Findings &amp; Insights</h2>

<!-- Organize findings by topic/question, not by repo -->
<!-- Use H3 for each major finding topic or objective being answered -->

<h3>{{Finding Topic / Objective Being Answered}}</h3>
<p>{{Prose summary of findings}}</p>

<!-- Payload/field comparison table when comparing producers or consumers -->
<table>
  <thead><tr><th>Field</th><th>Producer A</th><th>Producer B</th><th>Producer C</th><th>Notes</th></tr></thead>
  <tbody>
    <tr><td>field_name</td><td>✅</td><td>✅</td><td>❌</td><td>note</td></tr>
  </tbody>
</table>

<h3>{{Another Finding Topic}}</h3>
<p>{{Prose. Include inline GitHub links: <a href="https://github.com/Stodge-Inc/REPO/blob/main/PATH#Lnn">description</a>}}</p>

<hr>
<h2>Open Questions &amp; Further Research</h2>
<ul>
  <li>{{Question that still needs human input or follow-up — include any known partial answers inline}}</li>
</ul>
<hr>
<h2>Possible Solutions</h2>

<!-- Only include if research surfaced clear architectural options. Rough options only — not formal proposals. -->

<h3>Solution 1: {{Name}}</h3>
<p>{{Description of the approach}}</p>
<!-- Include payload tables for solutions that define a data contract -->
<p><strong>Payload:</strong></p>
<table>
  <thead><tr><th>Variable</th><th>Type</th><th>Required</th><th>Description</th></tr></thead>
  <tbody>
    <tr><td>field_name</td><td>string</td><td>yes</td><td>description</td></tr>
    <tr><td>optional_field</td><td>string</td><td>no</td><td>description</td></tr>
  </tbody>
</table>

<h3>Solution 2: {{Name}}</h3>
<p>{{Description}}</p>

<hr>
<h2>Other Considerations</h2>
<p>{{Approaches considered but deprioritized or rejected, and why.}}</p>
```

## Section Notes

- **Purpose & Scope**: Always include explicit In Scope / Out of Scope subsections.
- **Objectives framing**: Use "The technical design resulting from this discovery must answer the following questions:" — these drive the research agenda.
- **Objectives status**: ✅ = answered by research, 💬 = still open.
- **Findings**: Organized by topic/objective, not by repo. Each H3 answers a specific question or covers a major area. Synthesize across all repos — don't repeat per-repo output verbatim.
- **Payload comparison tables**: Use ✅/❌ per producer/consumer column. Essential when comparing how different services handle the same data.
- **Solution payload tables**: Use Variable / Type / Required / Description columns. These define the data contract.
- **Possible Solutions**: Rough options only. Skip if findings are purely informational.
- **GitHub links format**: `<a href="https://github.com/Stodge-Inc/{repo}/blob/main/{path}#L{line}">description</a>`
