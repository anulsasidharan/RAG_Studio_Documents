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
```bash
cd apps/web
rm -rf .next node_modules
npm install
npm run build
```

## Getting Help

- [GitHub Issues](https://github.com/yourusername/rag-studio/issues)
- [Discord](https://discord.gg/ragstudio)
- [Documentation](https://docs.ragstudio.io)
