<!-- the PUT said 200. the profile said otherwise. -->

<p align="center">
  <img src="./media/hero.png" alt="arr-stack-skills: agent skills for running a Radarr/Sonarr media stack" width="840">
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-00e5ff" alt="MIT license"></a>
  <a href="https://github.com/t3chnaztea/arr-stack-skills/releases"><img src="https://img.shields.io/github/v/release/t3chnaztea/arr-stack-skills?color=ff2e97" alt="Latest release"></a>
  <img src="https://img.shields.io/badge/skills-6-ffcc00" alt="6 skills">
  <img src="https://img.shields.io/badge/deps-none-8a2be2" alt="No dependencies">
  <img src="https://img.shields.io/badge/Claude%20Code-plugin-d97706" alt="Claude Code plugin">
</p>

<p align="center">
  <b>Point a coding agent at a real Radarr/Sonarr stack without learning every trap the expensive way, at 1am, with the queue at 400 and nothing importing.</b>
</p>

> The arr apps are well documented right up to the point where they stop
> behaving like the documentation. A cutoff that isn't a cap. A PUT that
> returns 200 and stores nothing. A root folder path that doesn't exist on the
> host. A profile setting whose default value guarantees infinite re-downloads.
> A delete that silently unmonitors the thing you deleted, so the re-grab you
> asked for matches zero episodes and reports success. Point an agent at this
> without doctrine and it will confidently tell you a job is done while your
> download client churns all night. This repo packages the doctrine as
> [Agent Skills](https://agentskills.io): focused, model-readable guides your
> coding agent loads on demand when you ask it to work on your media stack.

```
/plugin marketplace add t3chnaztea/arr-stack-skills
/plugin install arr@t3chnaztea-arr
```

## The six skills

**Start with `arr-connect`.** The others assume its API pattern, its path
model, and its read-after-write rule.

<table>
  <tr>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-connect/SKILL.md"><b>🔌 arr-connect</b></a><br />
      <sub>Start here. The v3 endpoint map (Sonarr v4 is still <code>/api/v3</code>), why <code>ls</code> on the host says your media folder doesn't exist when it's right there, why existence checks by title lie, and the read-modify-read loop that catches the PUTs which return 200 and change nothing.</sub>
    </td>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-context-map/SKILL.md"><b>🗺️ arr-context-map</b></a><br />
      <sub>Build the instance inventory this repo deliberately does not ship: what each profile id is actually <i>for</i>, which root folder pairs with which profile, container-to-host path mappings, and which app is template-managed. Without it an agent picks a profile by name and puts a 4K film on your phone tier.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-quality-profiles/SKILL.md"><b>🎚️ arr-quality-profiles</b></a><br />
      <sub>The trap that costs the most bandwidth: a <code>cutoffFormatScore</code> no release can reach means every item is permanently cutoff-unmet, forever. Plus why a cutoff is not a cap, why Sonarr rejects your cutoff id, and how a global release profile silently overrides the custom formats you just tuned.</sub>
    </td>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-library-hygiene/SKILL.md"><b>🧹 arr-library-hygiene</b></a><br />
      <sub>Three ways the <code>monitored</code> flag drifts from what you meant, including the one where deleting files auto-unmonitors them so your re-grab matches zero and reports success. The guarded delete-then-regrab sequence, and why you check the rejection reasons <i>before</i> anything is deleted.</sub>
    </td>
  </tr>
  <tr>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-downloads/SKILL.md"><b>📥 arr-downloads</b></a><br />
      <sub>Six links from "user wants a thing" to "file is in the library", and how to find which one broke instead of restarting everything. Empty search results versus a wall of rejected ones are opposite problems that look identical. Torrents at zero peers usually means your VPN region can't forward a port at all.</sub>
    </td>
    <td align="center" width="50%" valign="top">
      <a href="skills/arr-playback/SKILL.md"><b>📺 arr-playback</b></a><br />
      <sub>The loop most profile guides skip: what you grab decides whether your player transcodes. Image subtitles force a full 4K re-encode the moment someone turns subs on. Dolby Vision with no HDR10 fallback tone-maps. Your box may have no AV1 decoder. Tune the grab to the hardware, not to an ideal.</sub>
    </td>
  </tr>
</table>

---

## ⚠️ Read before you install

**These skills direct an agent to hold an API key that can delete your media
library.** Radarr and Sonarr ship with no recycle bin. A delete with
`deleteFiles=true` erases from disk immediately, with no trash folder and no
undo.

- **Review the skills before installing.** They are plain Markdown, no
  scripts, nothing that phones home or auto-runs. Read what they will have
  your agent do.
- **Turn on a recycle bin** in Settings > Media Management before you let an
  agent anywhere near a delete. It is the single cheapest insurance here.
- **The doctrine is conservative by design:** read before write, full-object
  PUT, verify from a fresh read rather than the write's own echo, confirm a
  replacement exists before deleting the file it would replace, and hand any
  bulk destructive pass to a human to read first. It is still your library and
  your risk.
- **Nothing here downloads anything or tells you where to get it.** These
  skills operate software you already run and configure. Sourcing content is
  your business and your jurisdiction's.
- **Keep the key out of the agent's context entirely.**
  [docs/secrets-hygiene.md](docs/secrets-hygiene.md) is the short version of
  how: the agent gets the capability, never the credential.

---

## Running alongside MCP servers

MCP servers for this stack exist and are multiplying:
[**bardesss/arr-mcp**](https://github.com/bardesss/arr-mcp) covers
Radarr/Sonarr/Prowlarr/Bazarr and friends from one server, and
[**dinglebear-ai/yarr**](https://github.com/dinglebear-ai/yarr) does the fleet
in Rust. If you want typed tools your agent can call instead of writing curl,
use one.

The split: **an MCP server gives your agent the API. This gives it the
playbook.** A tool called `update_quality_profile` will happily set
`cutoffFormatScore` to 10000 and hand you the upgrade storm described in
`arr-quality-profiles`; nothing in a tool schema knows that deleting a movie
silently unmonitors it, that a host-side `ls` lies about container paths, or
that a PUT can no-op and echo success. That judgment is transport-neutral: it
makes the agent better over curl and over MCP tools alike. Run both. They
compose.

---

## Install

### Claude Code plugin (recommended)

```
/plugin marketplace add t3chnaztea/arr-stack-skills
/plugin install arr@t3chnaztea-arr
```

The skills activate automatically when your prompt matches, e.g. "radarr keeps
re-downloading the same movie", "why does this 4K file stutter with subtitles
on", "sonarr says queued but nothing is happening", "the host says my media
folder doesn't exist".

### Manual copy (any Claude Code, no marketplace)

```bash
git clone https://github.com/t3chnaztea/arr-stack-skills
cp -r arr-stack-skills/skills/arr-* ~/.claude/skills/
```

### Other harnesses

Each `SKILL.md` is harness-neutral Markdown with standard Agent-Skills
frontmatter ([spec](https://agentskills.io/specification)). Drop the `skills/*`
directories wherever your agent framework discovers skills, or point it at this
repo.

### Configure

Nothing to install. Put the URL and key in your environment and the recipes
work as written:

```bash
export ARR_URL=http://arr-host:7878     # Radarr 7878, Sonarr 8989, Prowlarr 9696
export ARR_KEY=...                       # Settings > General > Security
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/system/status" | jq '{appName,version}'
```

If that prints an app name and version, you are ready. See
[`arr-connect`](skills/arr-connect/SKILL.md) for the rest.

---

## What's inside a skill

```
skills/arr-<area>/
  SKILL.md            # the guide (frontmatter + body, < 500 lines)
```

**No scripts, deliberately.** Everything is doctrine plus fenced curl recipes,
so your agent writes whatever throwaway code a task needs, fresh, against your
actual instance. A helper script would have to guess your paths, your profile
ids, and your container layout, and would rot the first time an endpoint moved.
Recipes stay honest.

Skills are original prose: operational lessons, not a copy of the manual. The
[Servarr wiki](https://wiki.servarr.com) and the
[TRaSH guides](https://trash-guides.info) remain the canonical references, and
they are excellent. These capture what running the stack with agents teaches
that those do not.

## Contributing

Have a hard-won arr lesson? Copy [`template/SKILL.md`](template/SKILL.md) and
follow its authoring notes: original prose, everything instance-specific
parameterized, **no real inventory of any kind** (no library contents, no
indexer names, no keys), state the app version you verified against, and show
the reader how to confirm the change rather than just make it. PRs welcome.

## More tiny tools for home labs

Agent skills: [unifi](https://github.com/t3chnaztea/unifi-skills) · [home-assistant](https://github.com/t3chnaztea/home-assistant-skills) · [batocera](https://github.com/t3chnaztea/batocera-skills) · [psn](https://github.com/t3chnaztea/awesome-psn-skills)  
Retro cabinet: [batocera-toolbox](https://github.com/t3chnaztea/batocera-toolbox) · [batocera-holidays](https://github.com/t3chnaztea/batocera-holidays)  
Home server: [dell-ipmi-fan-control](https://github.com/t3chnaztea/dell-ipmi-fan-control) · [plex-preroll-roulette](https://github.com/t3chnaztea/plex-preroll-roulette)  
PlayStation: [awesome-psnstats](https://github.com/t3chnaztea/awesome-psnstats)  
Desktop: [fastfetch-macos-gradient-hud](https://github.com/t3chnaztea/fastfetch-macos-gradient-hud)

## Versions

Verified against **Radarr v5** and **Sonarr v4** (both on API v3), mid-2026, in
a containerized deployment with a torrent client behind a VPN sidecar and a
Usenet client alongside it.

The arr apps move, and their companion ecosystem moves faster. Treat every
claim here as a strong prior, not gospel, and confirm against
`/api/v3/system/status` and your own `/api/v3/health` output. Where a trap is
version-scoped, the skill says so; where a behaviour has changed under you,
the fastest tell is usually the app's own health warnings, which almost nobody
reads.

## License

MIT; see [LICENSE](LICENSE).

> Not affiliated with the Servarr project, Ubiquiti, Plex, or any indexer.
> "Radarr", "Sonarr", "Prowlarr", "Plex", and "Jellyfin" are used
> descriptively; this is an independent, community-built collection. The apps
> live at [wiki.servarr.com](https://wiki.servarr.com).

---

The PUT said 200. The profile said otherwise. Read it back.
