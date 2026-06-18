# Kibo SDET Challenge — Quick Start

You've received the **Kibo SDET Technical Assessment** as a zip file.

## Getting Started

```bash
# 1. Unzip
unzip Kibo-SDET-Challenge-*.zip -d Kibo.SDET.Challenge
cd Kibo.SDET.Challenge

# 2. Initialize a git repo (for your submission)
git init
git add -A
git commit -m "Initial commit — assessment starter"

# 3. Restore & build
dotnet build

# 4. Start the Mock API (keep running in a separate terminal)
dotnet run --project src/Kibo.MockApi

# 5. Verify the legacy tests pass (in another terminal)
dotnet test tests/Kibo.LegacyTests
```

## What's Inside

| File / Folder | What It Is |
|---|---|
| `CANDIDATE_INSTRUCTIONS.md` | **Read this first.** Full task descriptions, API reference, and evaluation criteria. |
| `src/Kibo.MockApi/` | The mock API. **Do not modify.** |
| `src/Kibo.TestingFramework/` | Empty class library — build your testing SDK here. |
| `tests/Kibo.LegacyTests/` | Legacy tests to refactor. |
| `global.json` | Pins the .NET SDK version. |

## Requirements

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- An IDE (Visual Studio, VS Code, Rider)
- AI tools are encouraged — see instructions for details

## Submission

1. `dotnet build` succeeds with no errors
2. `dotnet test` passes
3. `PROMPT_LOG.md` is included in the repo root
4. Push to GitHub/GitLab and share the link, or zip and email

Full details in `CANDIDATE_INSTRUCTIONS.md`. Good luck!
