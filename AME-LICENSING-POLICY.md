# AME-FFMPEG — Proprietary Product and Third-Party Licensing Policy

**Product model:** AME-FFMPEG is intended to be a proprietary, closed-source product.  
**Status:** Initial engineering policy; not legal advice. A release-specific licence audit is still required.

## 1. Separate ownership

AME's original application code, user interface, orchestration layer, configuration model, APIs authored by the AME project, and product-specific features are intended to remain proprietary.

Third-party components retain their own licences. AME's proprietary status does not relicense FFmpeg or any dependency.

The product may therefore contain both:
- Proprietary AME code.
- Separately licensed third-party components distributed under their own licence terms.

## 2. FFmpeg integration policy

The preferred initial architecture is to dynamically link against FFmpeg shared libraries on Windows.

For an LGPL-based distribution, the release process must be designed to satisfy the applicable LGPL requirements. FFmpeg's official guidance currently recommends, among other things:

- Build without `--enable-gpl` and without `--enable-nonfree`.
- Use dynamic linking to FFmpeg libraries.
- Provide the exact corresponding FFmpeg source, including AME's FFmpeg modifications.
- Preserve and provide the applicable licence notices.
- Document the configure command and build process.
- Provide the source changes/patches.
- Do not prohibit reverse engineering of the FFmpeg library for debugging modifications.
- Do not obscure FFmpeg library names.
- Identify FFmpeg and its LGPL licence in the product's notices/About information and applicable distribution materials.

These are not optional branding preferences. The exact requirements must be checked against the LGPL version and distribution model selected for the release.

Official reference: https://ffmpeg.org/legal.html

## 3. Component policy

| Component | Initial policy | Reason |
|---|---|---|
| FFmpeg LGPL components | Candidate for use | Can be integrated into a proprietary application if applicable LGPL obligations are met |
| FFmpeg GPL components | Exclude | Would change FFmpeg's licensing obligations and conflict with the intended proprietary product model |
| x264 / libx264 | Exclude | GPL integration; not suitable for the planned LGPL-only build |
| Native FFmpeg AAC encoder | Candidate for use | Included in FFmpeg; verify exact source/configuration licensing during release audit |
| Intel QSV integration | Investigate and verify | Confirm current FFmpeg configure licensing status and external oneVPL dependency terms |
| NVIDIA NVENC integration | HOLD pending explicit review | Verify the current FFmpeg version's configure restrictions, ffnvcodec headers, NVIDIA runtime terms and redistribution model |
| Other external libraries | No automatic approval | Each dependency requires a licence and redistribution review |

**Do not assume that a component is safe merely because it compiles.** Build success is not a licence audit.

## 4. Source repository and product source are different things

The public AME-FFmpeg fork may contain FFmpeg's upstream source and any changes to that source that must be made available under the applicable licence.

That does not automatically require publishing the separate proprietary AME application source. Keep the boundary clear:

- FFmpeg fork/source distribution: follow FFmpeg's applicable licence.
- Proprietary AME application: retain closed source, subject to the rights and obligations of all linked components.
- Build scripts and notices: publish or provide whatever is required to satisfy the applicable third-party licences.

Do not put proprietary AME application source into the public FFmpeg fork unless intentionally choosing to publish it.

## 5. Release gate

No redistributable AME build should be approved until the release has:

1. A generated list of every enabled FFmpeg component.
2. A licence classification for each enabled component and external dependency.
3. A verified configure command and toolchain record.
4. The exact matching FFmpeg source archive/revision.
5. A patch set for every FFmpeg source modification.
6. Required licence texts and third-party notices.
7. A documented dynamic-linking/replacement approach consistent with the applicable LGPL.
8. A separate review of codec patent and other jurisdiction-specific issues.

FFmpeg itself warns that codec patent considerations can matter when distributing commercial software. This is a separate question from copyright licensing and needs jurisdiction-specific advice before commercial distribution.

## 6. Current decisions

- Product source model: **Proprietary / closed source**.
- FFmpeg strategy: **Use FFmpeg under its existing licence; do not attempt to relicense it**.
- Preferred integration: **Dynamic libraries**.
- GPL components: **Excluded by policy**.
- Nonfree components: **Not approved by default; explicit review required**.
- NVENC: **Not yet approved for the proprietary distribution build**.
- x264: **Excluded from the LGPL-only build**.

This document is a policy baseline, not a final legal opinion. Recheck it against the exact FFmpeg revision, configure output, dependency versions and intended distribution model before release.
