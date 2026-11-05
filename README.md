# pi-skill-code-review

[![Pi Package](https://img.shields.io/badge/pi--package-skill-blue)](https://github.com/IgorGuariroba/pi-skill-code-review)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**code-review** skill for the **[Pi coding agent](https://pi.dev)**, adapted from Matt Pocock's Engineering Skills.

> Review the changes since a fixed point (commit, branch, tag, or merge-base) along two axes: Standards (does the code follow this repo's documented coding standards?) and Spec (does the code match what the originating issue/spec asked for?). Runs independent Pi workers for both axes and reports them side by side. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to "review since X".

---

## Installation

Install directly into Pi using `pi install`:

```bash
pi install git:github.com/IgorGuariroba/pi-skill-code-review
```

Or clone into your local/global skills directory:

```bash
# Global Pi skills directory:
git clone https://github.com/IgorGuariroba/pi-skill-code-review.git ~/.pi/agent/skills/code-review

# Or Agent Skills standard path:
git clone https://github.com/IgorGuariroba/pi-skill-code-review.git ~/.agents/skills/code-review
```

---

## Usage

Trigger it explicitly in your session:

```text
/skill:code-review
```

Or let Pi automatically route to it based on conversation context and need.

---

## Credits

- **Original concept & methodology**: [Matt Pocock](https://github.com/mattpocock)
- **Adapted for Pi Agent**: [Igor Guariroba](https://github.com/IgorGuariroba)

---

## License

[MIT](LICENSE)
