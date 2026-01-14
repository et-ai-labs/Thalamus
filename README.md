# Thalamus Monorepo

This is the main Thalamus monorepo containing three submodules:

- **CLI** - Thalamus command-line interface
- **Server** - Thalamus backend server
- **App** - Thalamus application

## Project Structure

```
Thalamus/
├── CLI/           # Thalamus-CLI submodule
├── Server/        # Thalamus-Server submodule
├── App/           # Thalamus-App submodule
├── .gitmodules    # Submodule configuration
└── README.md      # This file
```

## Getting Started

### Initial Clone

To clone this repository with all submodules:

```bash
git clone --recurse-submodules https://github.com/Et-Ai-Labs/Thalamus.git
```

Or if you've already cloned without submodules:

```bash
git clone https://github.com/Et-Ai-Labs/Thalamus.git
cd Thalamus
git submodule update --init --recursive
```

### Updating Submodules

To pull the latest changes from all submodules:

```bash
git submodule update --remote --merge
```

To update a specific submodule:

```bash
cd CLI  # or Server, or App
git pull origin main
cd ..
git add CLI
git commit -m "Update CLI submodule"
```

## Working with Submodules

### Making Changes to a Submodule

1. Navigate to the submodule directory:
   ```bash
   cd CLI  # or Server, or App
   ```

2. Make your changes and commit them:
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin main
   ```

3. Return to the parent repository and update the submodule reference:
   ```bash
   cd ..
   git add CLI
   git commit -m "Update CLI submodule to latest commit"
   git push
   ```

### Pulling Changes from Others

When someone updates a submodule, you'll need to update your local copy:

```bash
git pull
git submodule update --init --recursive
```

## Submodule Repositories

- CLI: https://github.com/Et-Ai-Labs/Thalamus-CLI
- Server: https://github.com/Et-Ai-Labs/Thalamus-Server
- App: https://github.com/Et-Ai-Labs/Thalamus-App

## Important Notes

- Each submodule is a separate git repository with its own history
- Changes in submodules must be committed and pushed in both the submodule AND the parent repo
- Always run `git submodule update` after pulling changes to sync submodule commits
- The parent repo tracks specific commits of each submodule, not branches
