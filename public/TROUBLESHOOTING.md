# Troubleshooting Guide

## Common Issues

### Port Already in Use
```bash
# Change ports in docker-compose.yml
```

### Database Connection Failed
```bash
docker-compose down -v
docker-compose up -d
```

### API Not Responding
```bash
docker logs ragstudio-api
```

### Frontend Build Failed

Prefer installs and builds from the **repository root** so npm workspaces resolve correctly:

```bash
# From repo root (E:\...\Unified_RAG_Studio)
npm install
npm run build
```

To reset only the web app:

```bash
# From repo root
Remove-Item -Recurse -Force apps\web\.next -ErrorAction SilentlyContinue
npm install
npm run build --workspace=@rag-studio/web
```

### npm: `ENOWORKSPACES` — “This command does not support workspaces”

npm (v9+) may refuse commands such as `npm config get registry` when your shell’s **current directory** is inside a workspace package (for example `apps/web`), because it detects the workspace root above you.

**Fix:** run those commands from the **monorepo root**, not from `apps/web`:

```powershell
cd E:\EURON_AI_PRODUCT_ENG-2\Unified_RAG_Studio
npm config get registry
```

For installs and scripts, stay at the root and use `--workspace=@rag-studio/web` when you only need the web package:

```bash
npm install --workspace=@rag-studio/web
npm run dev --workspace=@rag-studio/web
```

## Getting Help

- [GitHub Issues](https://github.com/yourusername/rag-studio/issues)
- [Discord](https://discord.gg/ragstudio)
- [Documentation](https://docs.ragstudio.io)
