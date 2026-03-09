**Fix: TanStack + Storybook Port Conflict**

_Running both simultaneously on a Vite-based project_

# The Problem

When running TanStack Start and Storybook simultaneously in a Vite-based project, the following error is thrown:

| Error: listen EADDRINUSE: address already in use :::42069at ServerEventBus.start (.../@tanstack/devtools-event-bus/.../server.js)at BasicMinimalPluginContext.configureServer (.../@tanstack/devtools-vite/.../plugin.js) |
| --- |

The root cause: both TanStack Start's Vite config and Storybook's internal Vite instance load the same plugin set. The

The root cause: both TanStack Start's Vite config and Storybook's internal Vite instance load @tanstack/devtools-vite plugin's configureServer hook fires in both Vite instances, both trying to bind port 42069 for its event bus — the second one crashes.

## Conflicting Plugins (loaded in both Vite instances)

*   @tanstack/devtools:\* — owns port 42069, 6 sub-plugins
*   tanstack-react-start:\* — SSR/routing, meaningless in Storybook
*   nitro:\* — server bundler, meaningless in Storybook

# The Solution

Override Storybook's Vite config using the viteFinal hook in .storybook/main.ts. Instead of trying to filter the merged plugin list (unreliable due to nested plugin arrays), hard-replace config.plugins with only the plugins that are safe for a browser-only Storybook environment — then pull back Storybook's own internal plugins from the original config.

## Final Working Config

| // .storybook/main.tsimport type { StorybookConfig } from "@storybook/react-vite";import type { InlineConfig } from "vite";import viteReact from "@vitejs/plugin-react";import viteTsConfigPaths from "vite-tsconfig-paths";import tailwindcss from "@tailwindcss/vite";const config: StorybookConfig = {stories: ["../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],addons: ["@storybook/addon-docs","@storybook/addon-a11y","@storybook/addon-themes",],framework: "@storybook/react-vite",core: {enableCrashReports: false,disableWhatsNewNotifications: true,},async viteFinal(config: InlineConfig) {// Keep only Storybook's own internal plugins// @ts-ignoreconst storybookPlugins = (config.plugins ?? []).flat(Infinity).filter((plugin: any) => {if (!plugin?.name) return false;return (plugin.name.startsWith("storybook:") ||plugin.name === "plugin-csf");});// Hard-replace all plugins — never inherit TanStack/Nitro server pluginsconfig.plugins = [...storybookPlugins,viteTsConfigPaths({ projects: ['./tsconfig.json'] }),tailwindcss(),viteReact({ babel: { plugins: ['babel-plugin-react-compiler'] } }),];return config;},};export default config; |
| --- |

# Why This Works

viteFinal gives you the fully merged Vite config that Storybook has assembled, including all plugins inherited from your root vite.config.ts.

Instead of filtering (which silently fails when plugins are nested arrays or wrapped), we hard-replace config.plugins entirely.

We then pull back only the storybook: prefixed plugins from the original merged config — these are what actually render stories (CSF loader, HMR boundary, export order injection, etc.).

Finally we re-add the three Vite plugins that are safe in a browser-only context: path aliases, Tailwind, and React.

## Key Insight

Storybook spins up its own full Vite dev server — not a subset. Any plugin with a configureServer hook will fire in it. TanStack Devtools uses configureServer to bind a TCP port, so it must never run inside Storybook's Vite instance.

# How to Debug Plugin Names

If you update dependencies and the issue returns, log the exact plugin names Storybook sees:

| async viteFinal(config: InlineConfig) {const names = (config.plugins ?? []).flat(Infinity).map((p: any) => p?.name).filter(Boolean);console.log("[Storybook plugins]:", names);return config;} |
| --- |

Run pnpm storybook and check the terminal output. Update the filter list to match any new plugin names from TanStack, Nitro, or other server-side packages.