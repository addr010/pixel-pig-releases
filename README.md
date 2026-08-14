# Pixel Pig  
*A desktop studio for AI workflows*  

## Screenshots  
![Pixel Pig Main Window](pixel-pig-work.png)  

---

## 🎬 What is Pixel Pig?  
Pixel Pig is a standalone desktop app for Windows and macOS that helps you generate, organise, and manage AI-generated assets through a consistent workflow — no matter which API provider you use.  

It supports multiple API aggregators including **Fal**, **Replicate**, and **Kie.ai**, with more added based on community interest.  
Early support is also available for connecting to local **ComfyUI** instances for text-to-image batching and remote connections.  

Pixel Pig supports a wide range of AI workflows for creating and editing images, videos, voice, sound effects, lip-sync, and 3D assets.

Models that rank highly on public leaderboards are included out of the box, and **community model mappings** are supported so you can extend what’s available.

- Simple batch operations — run multiple models and aspect ratios in just a few clicks  
- No manual uploads or downloads — everything’s handled automatically  
- Built-in tools for file viewing, frame extraction, cropping, colour adjustment, format conversion, and batch management  
- Flexible Prompt Builder — customise categories of snippets, save favourites, and tag prompts for quick access  

---

## 🔍 Beta Release – Try It Now  
This repository hosts the **beta builds** of Pixel Pig.  

**[⬇️ Download Pixel Pig](https://www.pixelpig.net/)**

Pixel Pig is **donation-ware** — free to use.  
If you or your team are making money with it, consider *sharing the love*.  
Donation links are available inside the app.  

---

## 🛠 Getting Started  
1. **[Download the latest release](https://www.pixelpig.net/)** for your platform.  
2. Run the executable (Windows installer) or app bundle (macOS).  
3. Your OS will show a security warning — on Windows click "More info" then "Run anyway", on macOS click "Open" to confirm.  
4. Use it in your workflow and see how it fits.  
5. Found a bug or something odd? [Open an issue](../../issues).  
6. Training and how-to support are available in the [Pixel Pig Skool community](https://www.skool.com/pixel-pig-2056/classroom).  

**🌟 If Pixel Pig saves you time, consider [leaving a star on GitHub](https://github.com/addr010/pixel-pig-releases) — it helps others discover the project.**  

### 🤖 MCP for Local AI Clients

Pixel Pig releases include a bundled MCP server for local AI tools such as Codex, Claude Code, and other MCP-compatible clients.

- If Pixel Pig is already running, connect to `http://127.0.0.1:7361/mcp`
- If your AI client prefers `stdio`, launch:
  - macOS: `/Applications/PixelPig.app/Contents/Resources/app/PixelPig.McpServer --stdio`
  - Windows: `C:\Path\To\PixelPig\PixelPig.McpServer.exe --stdio`
- The app normally starts and stops the MCP sidecar automatically, so manual launch is only needed when your AI client wants to start the server itself.

Detailed setup and troubleshooting: [MCP-SETUP.md](MCP-SETUP.md)

Customer walkthroughs: [Codex](https://www.pixelpig.net/help/create-with-pixelpig-in-codex/) · [Claude Code](https://www.pixelpig.net/help/create-with-pixelpig-in-claude-code/)

### 🎬 Pixel Pig Agent Skills

Pixel Pig includes Cinema Director for people using Codex, Claude Code and other tools that support `SKILL.md`.

- `/pixelpig-cinema-director` is installed and kept current automatically when you open Pixel Pig. It turns a plain-language idea into an approved visual plan, then guides image, video, dialogue, sound, rough-cut, retake, and delivery work through Pixel Pig MCP.
- `/hyperframes-pixelpig` is optional. It prepares a Pixel Pig movie/project for the current HyperFrames workflow, preserves linked project media, verifies the finished render, and attaches it back to Pixel Pig.

Add the HyperFrames bridge only if you use HyperFrames:

```bash
npx skills add addr010/pixel-pig-releases --skill hyperframes-pixelpig
```

HyperFrames has its own skills for composition, animation, media preparation, validation, preview, and rendering. Its mandatory `hyperframes` entrypoint installs or refreshes the skills required for a particular composition.

Example prompts:

> Use Pixel Pig Cinema Director to help me turn this idea into a short cinematic scene. Plan it with me before generating anything.

> Using `/hyperframes-pixelpig`, begin a HyperFrames composition from my Pixel Pig movie, preview it for approval, then attach the verified finished render back to the movie.

The Pixel Pig skills need the Pixel Pig MCP server. The easy path is to open Pixel Pig and connect to `http://127.0.0.1:7361/mcp`; `stdio` clients can launch `PixelPig.McpServer --stdio` without keeping the app open. See [MCP-SETUP.md](MCP-SETUP.md).

### 🔧 FFmpeg

Pixel Pig now bundles FFmpeg for audio and video combine features. You should not need to install FFmpeg separately.

If the media combine tools do not appear in the app, please restart Pixel Pig and open an issue with your operating system and app version.

---

## 🎯 Known Issues & Roadmap  

**Known issues**  
- The macOS version currently receives more pre-release testing than the Windows build.  
- The app has not been tested on low-spec machines or older versions of macOS/Windows.  
- Limited model selection on Kie.ai and Replicate. If you urgently need a specific model, please open an issue.  

**What’s next**  
- Custom workflow presets — save and share  
- Pipelines — chain workflows for complex runs in one go  

Nightly builds can be made available if there’s interest.  

---

Thanks for giving Pixel Pig a go — I hope it earns a place in your toolkit.  
