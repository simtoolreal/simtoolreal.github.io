# SimToolReal Website Agent Notes

This repo is the project website plus a browser MuJoCo/WASM demo for the
SimToolReal policy. The website is mostly static HTML/CSS, but the MuJoCo demo
has several non-obvious moving parts. This file documents the parts most likely
to break when future agents edit the repo.

## Important Files

- `index.html`
  - Main project page.
  - Embeds the desktop MuJoCo demo in an iframe.
  - Hero/banner content, paper links, video sections, abstract, BibTeX.

- `demo.html`
  - Standalone responsive demo wrapper.
  - Desktop controls legend plus mobile overlay controls.
  - Uses `./mujoco_wasm/dist-desktop/index.html`.

- `demo-mobile.html`
  - Dedicated mobile demo wrapper.
  - Uses `./mujoco_wasm/dist-mobile/index.html`.

- `static/css/index.css`
  - Website styling, including mobile hero spacing.

- `mujoco_wasm/assets/scenes/iiwa_sharpa.xml`
  - The MuJoCo scene used by the browser demo.
  - Robot, table, object, goal object, joints, actuators, gravcomp, and object
    geometry live here.

- `mujoco_wasm/dist-desktop/assets/index-DwBApKdY.js`
- `mujoco_wasm/dist-mobile/assets/index-AavVMsXI.js`
- `mujoco_wasm/dist-mobile/assets/index-Biog2l8U.js`
  - Built/minified WASM demo bundles.
  - There is no human-readable source for these bundles in this checkout.
  - Edits here are fragile but currently necessary for website fixes.

## Local Preview

The repo can be served as static files:

```bash
cd /home/tylerlum/github_repos/simtoolreal.github.io
python3 -m http.server 8766 --bind 127.0.0.1
```

Useful local links:

- Main site: `http://127.0.0.1:8766/`
- Standalone wrapper: `http://127.0.0.1:8766/demo.html`
- Mobile wrapper: `http://127.0.0.1:8766/demo-mobile.html`
- Raw desktop WASM demo: `http://127.0.0.1:8766/mujoco_wasm/dist-desktop/`

## MuJoCo XML Loading and Cache Gotcha

The WASM demo writes files into MuJoCo's virtual `/working` filesystem before
calling:

```js
MjModel.loadFromXML("/working/iiwa_sharpa.xml")
```

There are two separate fetch paths for `iiwa_sharpa.xml` in each built bundle:

1. An explicit initial scene write:
   ```js
   fetch("../assets/scenes/" + yo + "?v=" + Date.now())
   ```

2. The asset preload loop:
   ```js
   fetch("../assets/scenes/" + I + (I.endsWith(".xml") ? "?v=" + Date.now() : ""))
   ```

This second path matters. We previously had only the first fetch cache-busted.
The preloader then fetched a stale cached XML and overwrote `/working/iiwa_sharpa.xml`,
which made the compiled MuJoCo model silently use old body `gravcomp` values.

Keep XML cache-busting on both paths. It is intentionally limited to `.xml`
files; meshes and other large assets still use normal browser caching.

## Gravity Compensation

Robot gravity compensation is defined in `iiwa_sharpa.xml`, not by runtime JS.

Current intended behavior:

- Robot bodies from `base` through all arm/hand/finger bodies have
  `gravcomp="1"`.
- `table`, `object`, and `goal_object` have no `gravcomp`, so they compile to
  `body_gravcomp = 0`.
- XML explicitly sets:
  ```xml
  <option ... gravity="0 0 -9.81"/>
  ```

Do not reintroduce runtime hacks like:

```js
model.body_gravcomp.fill(...)
```

That was useful for debugging but is not the final fix.

### Why `gravcomp` Is Explicit on Each Robot Body

Do not use this invalid default block:

```xml
<default class="iiwa">
  <body gravcomp="1"/>
</default>
```

Local MuJoCo rejected that with a schema error. The robust version is explicit
`gravcomp="1"` on the robot body tags themselves. This matches the effect of
programmatically setting `body.gravcomp = 1.0` on the robot spec bodies before
compilation, but keeps the website demo declarative.

### How to Verify Gravcomp

Use local Python MuJoCo from the `mjlab` environment if available:

```bash
cd /home/tylerlum/github_repos/mjlab
uv run python - <<'PY'
import mujoco
xml = "/home/tylerlum/github_repos/simtoolreal.github.io/mujoco_wasm/assets/scenes/iiwa_sharpa.xml"
model = mujoco.MjModel.from_xml_path(xml)
name_to_id = {
    mujoco.mj_id2name(model, mujoco.mjtObj.mjOBJ_BODY, i): i
    for i in range(model.nbody)
}
for name in ["base", "link1", "link7", "palmleft_hand_C_MC", "table", "object", "goal_object"]:
    i = name_to_id[name]
    print(name, i, model.body_gravcomp[i])
print("gravity", model.opt.gravity)
print("unique", sorted(set(float(x) for x in model.body_gravcomp)))
PY
```

Expected:

- `base`, `link1`, `link7`, `palmleft_hand_C_MC`: `1.0`
- `table`, `object`, `goal_object`: `0.0`
- unique body gravcomp values: `[0.0, 1.0]`
- gravity: `[0, 0, -9.81]`

## Policy, Joint Order, and Joint Limits

The browser policy logic is embedded in the built JS bundles. The policy uses
29 controlled joints:

- 7 IIWA arm joints first.
- 22 Sharpa hand joints after that.

The MuJoCo joint order in `iiwa_sharpa.xml` and the policy arrays in the JS
bundle must agree for:

- normalized joint-position observations,
- joint-velocity observations,
- previous target observations,
- action-to-PD target scaling.

The important arrays in the minified bundle are:

- `bE`: lower joint limits for the 29 policy joints.
- `jw`: upper joint limits for the 29 policy joints.
- `rc`: default joint positions.

The joint order is intended to match IsaacGym/SimToolReal policy order. The
bug we fixed previously was not a joint-order bug; it was stale/wrong pinky
joint limit scaling in the bundle. That affected both:

- input normalization of joint positions, and
- output action scaling to PD targets.

The corrected pinky limits in policy order are:

- `palmleft_pinky_CMC`: lower `0.0`, upper `0.2618`
- `palmleft_pinky_MCP_FE`: lower `-0.174533`, upper `1.5708`
- `palmleft_pinky_MCP_AA`: lower `-0.03491`, upper `0.03491`
- `palmleft_pinky_PIP`: lower `0.0`, upper `1.7453`
- `palmleft_pinky_DIP`: lower `0.0`, upper `1.3963`

If editing joint limits, update all three built bundles consistently.

## Policy Timing and Control

The browser demo uses:

- MuJoCo timestep from XML: `0.001`.
- Policy control target rate: approximately 60 Hz.
- `policyDecimation = round((1 / 60) / timestep)`.

Policy outputs are converted to position targets:

- Arm actions are delta targets with a speed scale.
- Hand actions are absolute joint targets blended with previous targets.

The actuators in `iiwa_sharpa.xml` are MuJoCo `general` actuators configured as
affine PD-like position servos. The browser writes `data.ctrl[i] =
prevTargets[i]` for the 29 policy joints.

## Object and Goal Geometry

The website demo currently uses a fixed tool-like primitive in
`iiwa_sharpa.xml`:

```xml
<body name="object" pos="0 0 0.545">
  <joint name="object_free_joint" type="free"/>
  <geom name="object_handle_geom" type="box" size="0.10 0.015 0.01" ... density="400"/>
  <geom name="object_head_geom" type="capsule" fromto="0.10 -0.03 0 0.10 0.03 0" size="0.02" ... density="300"/>
</body>
```

The goal is a mocap body:

```xml
<body name="goal_object" ... mocap="true">
  ...
</body>
```

Important gotchas:

- The physical object should not have `gravcomp`.
- The goal object is kinematic/mocap and should not be treated as a physical
  free object.
- The policy observation and keypoint logic in the bundle assumes a fixed
  object scale vector:
  ```js
  fw = [5, 0.75, 0.5]
  ```
- This keypoint scale is policy-specific and should match the pretrained
  policy/config assumptions. Do not casually change object dimensions without
  revisiting policy observations, keypoints, and success/reward computations.

## Object/Goal Frame Axes

The raw WASM demo supports diagnostic axes for both the physical object and
goal object.

- Press `X` to toggle object/goal axes.
- Axes are hidden by default.
- The wrappers show desktop controls for `X`.
- Mobile wrapper intentionally no longer exposes a toggle-axes button.

The axes are useful when checking whether the goal visualization is offset from
the object pose. The object pose origin should be interpreted relative to the
body frame used in MuJoCo/policy keypoints, not the visual center of the entire
combined handle/head geometry.

## Goal and Object Manual Controls

Desktop/raw WASM controls include:

- Arrow keys: move goal left/right/up/down.
- `[` / `]`: move goal forward/back in Y.
- `R`: randomize goal pose.
- `W` / `A` / `S` / `D`: teleport object on the table.
- `Backspace`: reset simulation.
- `X`: toggle object/goal axes.

Mobile wrappers currently expose only:

- arrow D-pad,
- `Random Goal`,
- `Reset Sim`.

## Camera

The current raw WASM camera setup in the bundle uses:

```js
camera.position.set(0, 1.03, 1)
controls.target.set(0, 0.71, 0)
```

The position was kept fixed while the target was raised to frame the robot and
table better. If changing camera framing, check both desktop and mobile.

## Editing Built Bundles

The `dist-*` JS files are minified built artifacts. There is no readable source
in this repo for the custom policy/viewer code. When changing them:

1. Make the smallest possible string replacement.
2. Apply the same change to all three bundles:
   - desktop,
   - mobile `AavVMsXI`,
   - mobile `Biog2l8U`.
3. Run syntax checks:
   ```bash
   node --check mujoco_wasm/dist-desktop/assets/index-DwBApKdY.js
   node --check mujoco_wasm/dist-mobile/assets/index-AavVMsXI.js
   node --check mujoco_wasm/dist-mobile/assets/index-Biog2l8U.js
   ```
4. Search for debug leftovers before committing:
   ```bash
   rg "console.table|body_gravcomp.fill|gravtest|disableflags|window.__simtoolrealDemo" mujoco_wasm/dist-*/assets/*.js
   ```

Debug hooks are fine locally, but do not commit them unless intentionally
exposing a supported debug feature.

## Common Failure Modes

- Robot arm appears to sag/fall in browser:
  - Check the compiled `body_gravcomp` values.
  - Most likely stale XML or missing `gravcomp` on child robot bodies.
  - Make sure both XML fetch paths cache-bust `.xml`.

- Object floats:
  - Something gave the object `body_gravcomp = 1`.
  - Ensure runtime JS is not doing `body_gravcomp.fill(1)`.
  - Ensure XML has no `gravcomp` on `object`.

- Policy behavior is poor after joint-limit edits:
  - Check policy lower/upper arrays (`bE`, `jw`) in all bundles.
  - Especially verify pinky order and limits.
  - Joint order itself should match policy order; do not reorder XML joints
    without updating observations/actions.

- Goal visually seems offset:
  - Toggle axes with `X`.
  - Confirm whether comparing body frame origin, visual mesh center, or
    keypoint-derived object center. These are not necessarily the same.

- Changes do not appear after reload:
  - Hard refresh the page.
  - Verify XML fetch cache-busting is present in both fetch paths.
  - The MuJoCo model is loaded from `/working/iiwa_sharpa.xml`, not directly
    from the network URL.

