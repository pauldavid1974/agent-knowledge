# autodesk-fusion-mcp

Driving Autodesk Fusion (desktop) from an AI agent through Autodesk's own MCP server. Started as
web research; the items marked **hands-on** were confirmed by driving Fusion from Claude Code.
Everything not marked is still from Autodesk's docs and third-party write-ups only.

**What this was true against:** Fusion desktop `2705.1.30`, Personal Use licence, Windows 11,
Claude Code. Hands-on checks run 2026-10.
**Check before trusting the dated parts:** in a connected agent, run a script that prints
`adsk.core.Application.get().version` (see the recipe below). Without an agent: Fusion's
Preferences > General > API should show a "Fusion MCP Server" checkbox; with it ticked,
`http://127.0.0.1:27182/mcp` answers.

---

## Durable — how it fits together

### Two servers. Don't mix them up

- **Fusion MCP Server (local).** Runs inside Fusion desktop and needs Fusion open. It drives
  modelling, runs commands and inspects the active document. It does not work with the web
  client. It listens on 127.0.0.1 only, port 27182 by default, with **no authentication**: any
  local program can drive it while it is enabled.
- **Fusion Data MCP Server (cloud).** Queries hubs, projects, folders and items through Autodesk
  cloud APIs, using OAuth. It does not need Fusion running. (Not tested hands-on.)

### Connecting Claude Code — hands-on, works

```
claude mcp add --transport http fusion http://127.0.0.1:27182/mcp
```

Autodesk's docs list Claude Desktop, Cursor and VS Code plus "any Streamable HTTP client";
Claude Code isn't named, but the connection works. Tick the checkbox in Fusion's
Preferences > General > API first. The Personal Use (hobbyist) licence includes it — confirmed
on a Personal install, where the checkbox reads "Fusion MCP Server (runs locally on this
device)" next to the port setting.

### The real workhorse is the script tool

Tools are discovered when the client connects, and the set changes between releases (see
*Perishable*). Whatever the high-level tools do, the one that makes anything possible is the
**script** execution: arbitrary Fusion API Python, run inside Fusion. Hands-on:

- The script defines `def run(_context: str):`. Whatever it `print()`s comes back as the
  result. **Don't catch exceptions** — the error text is how you learn what failed.
- **Units are centimetres** throughout the API. 2 mm is `0.2`.
- A `readOnly: true` flag marks a script that only looks. It is enforced (writes raise "Cannot
  modify the design from a read-only context"), and in exchange it runs even while the user has
  a command dialog open, which a normal script cannot. Setting the **camera** counts as
  read-only — that works.
- **Script-made features land in Fusion's timeline with normal names** (Sketch1, Extrude1,
  Fillet1...), editable by hand afterwards like any feature the user made. This was the big
  open question in the research; the answer is yes, at least with history (parametric) on.
- Check the design's up-axis before assuming XY is the floor:
  `app.preferences.generalPreferences.defaultModelingOrientation` (Z-up vs Y-up).

```python
import adsk.core, adsk.fusion

def run(_context: str):
    app = adsk.core.Application.get()
    print(app.version)
    design = adsk.fusion.Design.cast(app.activeProduct)
    root = design.rootComponent
```

### Recipes — hands-on, all worked

**Scratch document** — experiment without touching the user's real file:

```python
doc = app.documents.add(adsk.core.DocumentTypes.FusionDesignDocumentType)
design = adsk.fusion.Design.cast(app.activeProduct)   # now points at the scratch doc
# ... later:
doc.close(False)   # False = discard, no save prompt
```

**User parameters**, then use them by name in any feature value:

```python
design.userParameters.add('wall', adsk.core.ValueInput.createByString('2 mm'), 'mm', 'wall thickness')
adsk.core.ValueInput.createByString('wall')
```

**Measure** between two faces/edges/bodies (result in cm):

```python
r = app.measureManager.measureMinimumDistance(entity_a, entity_b)
print(r.value)
```

**Framed screenshot.** Fit the view first or the part is a speck in a corner; fit alone is
too tight, so widen it afterwards. Runs as `readOnly: true`:

```python
vp = app.activeViewport
cam = vp.camera
cam.isSmoothTransition = False
cam.viewOrientation = adsk.core.ViewOrientations.IsoTopRightViewOrientation
cam.isFitView = True
vp.camera = cam
cam = vp.camera                    # re-read: fit has changed it
cam.isSmoothTransition = False
cam.viewExtents = cam.viewExtents * 1.6
vp.camera = cam
```

Then take the screenshot with the server's read/screenshot query at the current view. A dark
background reads better than a transparent one. For the underside, use
`IsoBottomRightViewOrientation`.

### Safety

- Every script runs Python with the user's own permissions. There is no undo on the AI side
  beyond Fusion's own; an undo operation exists, but how much of a multi-feature script one undo
  removes is untested — check the timeline after.
- Don't save the user's document unless they ask. Use a scratch document for experiments.

---

## Perishable

### Tool names as exposed to Claude Code
**True against:** Fusion `2705.1.30`. **Check:** the tool list after connecting.

Registered as server `fusion`, the tools appeared as `fusion_mcp_read`, `fusion_mcp_execute`
(script, document open/close/save, plus image/presentation extras), `fusion_mcp_update`
(includes undo) and `fusion_mcp_electronics_read`. Expect this to change between releases.

### Reported strengths and weaknesses — not tested hands-on
**True against:** web write-ups from April–October 2026.

Autodesk shipped the integration in April 2026, marketed around Claude. Reported good at:
edits to existing models (fillets, chamfers, parameters), simple prismatic parts, reading and
summarising a model, writing add-ins. Reported weak at: organic shapes, lofts, surfaces, fully
constrained sketches, and recovering from failed features. The hands-on use so far (prismatic
printable parts via scripts) agrees with the first list; the second list was not exercised.

## Sources
- https://help.autodesk.com/view/fusion360/ENU/?guid=FMCP-OVERVIEW
- https://help.autodesk.com/view/ADSKMCP/ENU/?guid=ADSKMCP_FusionDesktopMcp_autodesk_fusion_mcp_server_html
- https://techdrawai.com/blog/connect-claude-to-fusion-360
