# 🤖 OrcBot Skills Vault

> Elevate your OrcBot agents with specialized intelligence, deterministic scripts, and advanced reasoning frameworks.

This repository is a curated collection of **OrcBot Skills**, designed to extend the capabilities of your AI agents through the [agentskills.io](https://agentskills.io) specification.

---

## 📂 Repository Structure

Each folder within `skills/` is a self-contained module that OrcBot can ingest to learn new workflows, tools, and domain expertise.

```bash
.
├── README.md
└── skills/
    ├── [skill-name]/
    │   ├── SKILL.md          # 🧠 Instructions, metadata, and triggers
    │   ├── scripts/          # ⚙️ Deterministic tools (Node.js/Python/Bash)
    │   ├── references/       # 📚 Domain-specific documentation
    │   └── assets/           # 🎨 Templates and static resources
```

---

## 📦 Installation for OrcBot

You can install any skill from this repository directly into your OrcBot instance using the methods below.

### 💬 Method 1: Direct Chat (Recommended)
Simply send a message to your OrcBot on Telegram, Discord, or WhatsApp providing the URL to the skill:

> **"Install this skill: https://github.com/your-username/your-skills-repo/tree/main/skills/your-skill-name"**

OrcBot will automatically clone the repository, install the skill, and activate its capabilities immediately.

### 💻 Method 2: Command Line (CLI)
If you have access to the server where OrcBot is running, use the following command:

```bash
orcbot skill install https://github.com/your-username/your-skills-repo/tree/main/skills/your-skill-name
```

### 📦 Method 3: NPM (For published skills)
If the skills are published to NPM following the `agentskills.io` specification, use:

```bash
orcbot skill install npm:your-package-name
```

---

## 🛠 Supported URL Formats
OrcBot's `install_skill` tool is highly flexible and supports:
* **GitHub Repository URLs**: Clones the entire repo and discovers all `SKILL.md` files.
* **Subdirectory URLs**: Target a specific skill within a mono-repo.
* **Gist URLs**: Install single-file skills hosted on GitHub Gists.
* **Raw File URLs**: Point directly to a raw `SKILL.md` file.

---

## ⚡ Note on Permissions
Since installing new skills can introduce executable code, this action is **Admin-only**. Ensure you are listed in the `adminUsers` section of your `orcbot.config.yaml` to trigger installations via chat.

---

## 🛠 Contributing a New Skill
1. **Fork** the repository.
2. Create a new directory in `skills/`.
3. Add a `SKILL.md` with required YAML frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Clear, concise description of when OrcBot should trigger this skill.
   ---
   ```
4. **Submit a Pull Request**.
