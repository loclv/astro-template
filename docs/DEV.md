# Development

## Running the development server

```bash
bun dev
# building for production
bun run build
# reviewing the production build
bun run preview
```

When starting the dev server, use background mode:

```sh
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Dependencies

Keep TypeScript version 6.x for compatibility with Astro's "@astrojs/check".
If update to TypeScript 7.x, "@astrojs/check" throws errors.
