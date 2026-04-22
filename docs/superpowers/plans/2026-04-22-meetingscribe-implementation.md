# MeetingScribe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a stateless Next.js 15 app that extracts a structured JSON summary from a pasted meeting transcript using Claude or OpenAI, streaming partial JSON to the UI in real time.

**Architecture:** Edge API route streams SSE; one JSON Schema drives two LLM adapters (Anthropic tool-calling, OpenAI strict structured outputs); client hook consumes the SSE and renders fields progressively; no database, no auth.

**Tech Stack:** Next.js 15 (App Router, Edge runtime), TypeScript 5 strict, Tailwind CSS, shadcn/ui, Zod, `@anthropic-ai/sdk`, `openai`, `partial-json`, `@upstash/ratelimit` + `@upstash/redis`, Vitest, Biome, pnpm.

**Reference spec:** [docs/superpowers/specs/2026-04-22-meetingscribe-design.md](../specs/2026-04-22-meetingscribe-design.md)

---

## File structure (summary)

```
.github/workflows/ci.yml
.env.example
.gitignore
biome.json
package.json
tsconfig.json
next.config.ts
postcss.config.mjs
tailwind.config.ts
components.json                     (shadcn config)
vitest.config.ts
LICENSE
README.md                           (rewritten, later task)

public/
  favicon.ico

src/
  app/
    layout.tsx
    globals.css
    page.tsx                        (landing)
    app/page.tsx                    (tool)
    api/summarize/route.ts          (Edge, SSE)
  components/
    ui/                             (shadcn primitives)
    Header.tsx
    landing/
      Hero.tsx
      WhyStructured.tsx
      JsonPreview.tsx
    app/
      TranscriptInput.tsx
      SamplePicker.tsx
      SummarizeButton.tsx
      ModelPicker.tsx
      KeyDialog.tsx
      SummaryCard.tsx
      SummaryField.tsx
      DecisionsList.tsx
      ActionItemsList.tsx
      ActionItemRow.tsx
      OpenQuestionsList.tsx
      ExportMenu.tsx
      StreamingCaret.tsx
  lib/
    llm/
      schema.ts                     (JSON Schema + Zod, source of truth)
      prompt.ts                     (shared system prompt)
      types.ts                      (LLMAdapter interface, StreamChunk, StreamResult)
      anthropic.ts                  (Claude adapter)
      openai.ts                     (OpenAI adapter)
      index.ts                      (pickAdapter)
    hooks/
      useSummaryStream.ts
      useLocalKeys.ts
    samples/
      product-kickoff.ts
      eng-standup.ts
      customer-call.ts
      index.ts                      (sample registry)
    rate-limit.ts
    validate.ts                     (Zod for request body)
    export.ts                       (toMarkdown)
    utils.ts                        (cn)
  lib/__tests__/
    schema.test.ts
    partial-json.test.ts
    anthropic.test.ts
    openai.test.ts
    pick-adapter.test.ts
    validate.test.ts
    export.test.ts
    fixtures/
      golden/
        product-kickoff.json
        eng-standup.json
        customer-call.json
      streams/
        anthropic-product-kickoff.txt
        openai-product-kickoff.txt
```

---

## Task 1: Scaffold Next.js 15 + pnpm + TypeScript strict

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `next.config.ts`
- Create: `.gitignore`
- Create: `src/app/layout.tsx`
- Create: `src/app/page.tsx` (temporary placeholder)
- Create: `src/app/globals.css` (empty, populated in Task 3)

- [ ] **Step 1: Create package.json**

```json
{
  "name": "meetingscribe",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=20",
    "pnpm": ">=9"
  },
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "next": "15.0.3",
    "react": "19.0.0",
    "react-dom": "19.0.0"
  },
  "devDependencies": {
    "@types/node": "22.7.4",
    "@types/react": "19.0.0",
    "@types/react-dom": "19.0.0",
    "typescript": "5.6.3"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "es2022"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "allowJs": false,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "incremental": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "plugins": [{ "name": "next" }]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules", ".next", "dist"]
}
```

- [ ] **Step 3: Create next.config.ts**

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  reactStrictMode: true,
  experimental: {
    typedRoutes: true,
  },
};

export default nextConfig;
```

- [ ] **Step 4: Create .gitignore**

```
node_modules
.next
.vercel
.env
.env.local
.env.*.local
dist
coverage
*.log
.DS_Store
```

- [ ] **Step 5: Create placeholder src/app/layout.tsx**

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "MeetingScribe",
  description: "Parseable meeting summaries. Not prose.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

- [ ] **Step 6: Create placeholder src/app/page.tsx**

```tsx
export default function Home() {
  return <main>MeetingScribe</main>;
}
```

- [ ] **Step 7: Create empty src/app/globals.css**

Create the file with zero bytes. Contents are added in Task 3.

- [ ] **Step 8: Install and verify**

Run: `pnpm install && pnpm typecheck && pnpm build`
Expected: install succeeds, typecheck passes, `.next/` built.

- [ ] **Step 9: Commit**

```bash
git add .
git commit -m "chore: scaffold Next.js 15 app with pnpm and strict TS"
```

---

## Task 2: Biome, Vitest, dev tooling

**Files:**
- Create: `biome.json`
- Create: `vitest.config.ts`
- Modify: `package.json` (pnpm add adds deps automatically)

- [ ] **Step 1: Add dev dependencies**

```bash
pnpm add -D @biomejs/biome@1.9.4 vitest@2.1.4 @vitest/coverage-v8@2.1.4
```

- [ ] **Step 2: Create biome.json**

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": false, "ignore": [".next", "coverage", "dist", "pnpm-lock.yaml"] },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2, "lineWidth": 100 },
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "style": { "noNonNullAssertion": "off" },
      "suspicious": { "noExplicitAny": "warn" }
    }
  },
  "javascript": { "formatter": { "quoteStyle": "double", "semicolons": "always", "trailingCommas": "all" } }
}
```

- [ ] **Step 3: Create vitest.config.ts**

```ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    environment: "node",
    include: ["src/**/__tests__/**/*.test.ts"],
    coverage: { reporter: ["text", "html"], exclude: ["src/app/**", "src/components/**"] },
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

- [ ] **Step 4: Verify lint and test runners**

Run: `pnpm lint`
Expected: no errors.

Run: `pnpm test`
Expected: "no test files found" — exits 0 with a note.

- [ ] **Step 5: Commit**

```bash
git add biome.json vitest.config.ts package.json pnpm-lock.yaml
git commit -m "chore: add Biome and Vitest"
```

---

## Task 3: Tailwind + fonts + dark theme tokens

**Files:**
- Create: `postcss.config.mjs`
- Create: `tailwind.config.ts`
- Modify: `src/app/globals.css`
- Modify: `src/app/layout.tsx`

- [ ] **Step 1: Add Tailwind and helpers**

```bash
pnpm add -D tailwindcss@3.4.14 postcss@8.4.47 autoprefixer@10.4.20 tailwindcss-animate@1.0.7
pnpm add clsx@2.1.1 tailwind-merge@2.5.4
```

- [ ] **Step 2: Create postcss.config.mjs**

```js
export default {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};
```

- [ ] **Step 3: Create tailwind.config.ts**

```ts
import type { Config } from "tailwindcss";
import animate from "tailwindcss-animate";

const config: Config = {
  darkMode: "class",
  content: ["./src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      fontFamily: {
        sans: ["var(--font-inter)", "system-ui", "sans-serif"],
        mono: ["var(--font-mono)", "ui-monospace", "monospace"],
      },
      colors: {
        accent: {
          DEFAULT: "#34d399",
          foreground: "#052e1a",
        },
      },
      keyframes: {
        blink: { "0%, 100%": { opacity: "1" }, "50%": { opacity: "0" } },
      },
      animation: { blink: "blink 1s step-end infinite" },
    },
  },
  plugins: [animate],
};

export default config;
```

- [ ] **Step 4: Populate src/app/globals.css**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  html {
    color-scheme: dark;
  }
  body {
    @apply bg-zinc-950 text-zinc-100 antialiased font-sans;
  }
  ::selection {
    @apply bg-accent/30 text-accent;
  }
}
```

- [ ] **Step 5: Wire fonts in src/app/layout.tsx**

```tsx
import type { Metadata } from "next";
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"], variable: "--font-inter" });
const mono = JetBrains_Mono({ subsets: ["latin"], variable: "--font-mono" });

export const metadata: Metadata = {
  title: "MeetingScribe — Parseable meeting summaries",
  description:
    "Upload a transcript; get structured JSON with decisions, action items, and open questions — guaranteed to match a schema.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`dark ${inter.variable} ${mono.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

- [ ] **Step 6: Verify**

Run: `pnpm build`
Expected: build succeeds; Tailwind compiles without warnings.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "chore: add Tailwind with dark theme and font variables"
```

---

## Task 4: shadcn/ui primitives + utils

**Files:**
- Create: `components.json`
- Create: `src/lib/utils.ts`
- Create: `src/components/ui/button.tsx`
- Create: `src/components/ui/textarea.tsx`
- Create: `src/components/ui/dialog.tsx`
- Create: `src/components/ui/select.tsx`
- Create: `src/components/ui/badge.tsx`

- [ ] **Step 1: Install primitive deps**

```bash
pnpm add @radix-ui/react-dialog@1.1.2 @radix-ui/react-select@2.1.2 @radix-ui/react-slot@1.1.0 class-variance-authority@0.7.0 lucide-react@0.453.0
```

- [ ] **Step 2: Create components.json**

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/app/globals.css",
    "baseColor": "zinc",
    "cssVariables": false,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/lib/hooks"
  }
}
```

- [ ] **Step 3: Create src/lib/utils.ts**

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

- [ ] **Step 4: Create src/components/ui/button.tsx**

```tsx
import * as React from "react";
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-accent disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        primary:
          "bg-accent text-accent-foreground hover:bg-accent/90 shadow-[0_0_0_1px_rgba(52,211,153,0.25),0_8px_24px_-8px_rgba(52,211,153,0.35)]",
        ghost: "hover:bg-zinc-900 text-zinc-200 border border-zinc-800",
        link: "text-accent underline-offset-4 hover:underline",
      },
      size: { sm: "h-8 px-3", md: "h-9 px-4", lg: "h-11 px-6 text-base" },
    },
    defaultVariants: { variant: "primary", size: "md" },
  },
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button";
    return <Comp ref={ref} className={cn(buttonVariants({ variant, size }), className)} {...props} />;
  },
);
Button.displayName = "Button";
```

- [ ] **Step 5: Create src/components/ui/textarea.tsx**

```tsx
import * as React from "react";
import { cn } from "@/lib/utils";

export const Textarea = React.forwardRef<
  HTMLTextAreaElement,
  React.TextareaHTMLAttributes<HTMLTextAreaElement>
>(({ className, ...props }, ref) => (
  <textarea
    ref={ref}
    className={cn(
      "flex min-h-[12rem] w-full resize-y rounded-md border border-zinc-800 bg-zinc-900 px-3 py-2 text-sm font-mono placeholder:text-zinc-600 focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-accent disabled:opacity-50",
      className,
    )}
    {...props}
  />
));
Textarea.displayName = "Textarea";
```

- [ ] **Step 6: Create src/components/ui/dialog.tsx**

```tsx
"use client";
import * as React from "react";
import * as DialogPrimitive from "@radix-ui/react-dialog";
import { X } from "lucide-react";
import { cn } from "@/lib/utils";

export const Dialog = DialogPrimitive.Root;
export const DialogTrigger = DialogPrimitive.Trigger;
export const DialogClose = DialogPrimitive.Close;

export const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <DialogPrimitive.Portal>
    <DialogPrimitive.Overlay className="fixed inset-0 z-50 bg-black/70 backdrop-blur-sm data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0" />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        "fixed left-1/2 top-1/2 z-50 grid w-full max-w-md -translate-x-1/2 -translate-y-1/2 gap-4 border border-zinc-800 bg-zinc-950 p-6 shadow-lg rounded-lg",
        className,
      )}
      {...props}
    >
      {children}
      <DialogPrimitive.Close className="absolute right-4 top-4 text-zinc-500 hover:text-zinc-100">
        <X className="h-4 w-4" />
        <span className="sr-only">Close</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPrimitive.Portal>
));
DialogContent.displayName = "DialogContent";

export const DialogHeader = ({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) => (
  <div className={cn("flex flex-col space-y-1.5", className)} {...props} />
);

export const DialogTitle = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Title>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Title>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Title
    ref={ref}
    className={cn("text-lg font-semibold leading-none", className)}
    {...props}
  />
));
DialogTitle.displayName = "DialogTitle";

export const DialogDescription = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Description>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Description>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Description
    ref={ref}
    className={cn("text-sm text-zinc-400", className)}
    {...props}
  />
));
DialogDescription.displayName = "DialogDescription";
```

- [ ] **Step 7: Create src/components/ui/select.tsx**

```tsx
"use client";
import * as React from "react";
import * as SelectPrimitive from "@radix-ui/react-select";
import { Check, ChevronDown } from "lucide-react";
import { cn } from "@/lib/utils";

export const Select = SelectPrimitive.Root;
export const SelectValue = SelectPrimitive.Value;

export const SelectTrigger = React.forwardRef<
  React.ElementRef<typeof SelectPrimitive.Trigger>,
  React.ComponentPropsWithoutRef<typeof SelectPrimitive.Trigger>
>(({ className, children, ...props }, ref) => (
  <SelectPrimitive.Trigger
    ref={ref}
    className={cn(
      "flex h-9 w-full items-center justify-between rounded-md border border-zinc-800 bg-zinc-900 px-3 py-2 text-sm focus:outline-none focus:ring-1 focus:ring-accent",
      className,
    )}
    {...props}
  >
    {children}
    <SelectPrimitive.Icon asChild>
      <ChevronDown className="h-4 w-4 opacity-50" />
    </SelectPrimitive.Icon>
  </SelectPrimitive.Trigger>
));
SelectTrigger.displayName = "SelectTrigger";

export const SelectContent = React.forwardRef<
  React.ElementRef<typeof SelectPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof SelectPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <SelectPrimitive.Portal>
    <SelectPrimitive.Content
      ref={ref}
      className={cn(
        "relative z-50 min-w-[8rem] overflow-hidden rounded-md border border-zinc-800 bg-zinc-950 text-zinc-100 shadow-md",
        className,
      )}
      position="popper"
      {...props}
    >
      <SelectPrimitive.Viewport className="p-1">{children}</SelectPrimitive.Viewport>
    </SelectPrimitive.Content>
  </SelectPrimitive.Portal>
));
SelectContent.displayName = "SelectContent";

export const SelectItem = React.forwardRef<
  React.ElementRef<typeof SelectPrimitive.Item>,
  React.ComponentPropsWithoutRef<typeof SelectPrimitive.Item>
>(({ className, children, ...props }, ref) => (
  <SelectPrimitive.Item
    ref={ref}
    className={cn(
      "relative flex w-full cursor-default select-none items-center rounded-sm py-1.5 pl-8 pr-2 text-sm outline-none focus:bg-zinc-900",
      className,
    )}
    {...props}
  >
    <span className="absolute left-2 flex h-3.5 w-3.5 items-center justify-center">
      <SelectPrimitive.ItemIndicator>
        <Check className="h-4 w-4" />
      </SelectPrimitive.ItemIndicator>
    </span>
    <SelectPrimitive.ItemText>{children}</SelectPrimitive.ItemText>
  </SelectPrimitive.Item>
));
SelectItem.displayName = "SelectItem";
```

- [ ] **Step 8: Create src/components/ui/badge.tsx**

```tsx
import { cn } from "@/lib/utils";

export function Badge({
  className,
  variant = "default",
  ...props
}: React.HTMLAttributes<HTMLDivElement> & { variant?: "default" | "accent" | "muted" }) {
  return (
    <div
      className={cn(
        "inline-flex items-center rounded-sm border px-2 py-0.5 text-xs font-medium",
        variant === "default" && "border-zinc-800 bg-zinc-900 text-zinc-300",
        variant === "accent" && "border-accent/30 bg-accent/10 text-accent",
        variant === "muted" && "border-transparent bg-transparent text-zinc-500",
        className,
      )}
      {...props}
    />
  );
}
```

- [ ] **Step 9: Verify**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 10: Commit**

```bash
git add .
git commit -m "feat: add shadcn/ui primitives (button, textarea, dialog, select, badge)"
```

---

## Task 5: JSON Schema + Zod schema (TDD)

**Files:**
- Create: `src/lib/llm/schema.ts`
- Create: `src/lib/__tests__/schema.test.ts`

- [ ] **Step 1: Install Zod**

```bash
pnpm add zod@3.23.8
```

- [ ] **Step 2: Write failing test — src/lib/__tests__/schema.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { SummarySchema, summaryJsonSchema } from "@/lib/llm/schema";

describe("summaryJsonSchema", () => {
  it("requires all four top-level fields for strict OpenAI compatibility", () => {
    expect(summaryJsonSchema.required).toEqual([
      "summary",
      "decisions",
      "action_items",
      "open_questions",
    ]);
    expect(summaryJsonSchema.additionalProperties).toBe(false);
  });

  it("requires task, owner, and due_date inside action_items for strict mode", () => {
    expect(summaryJsonSchema.properties.action_items.items.required).toEqual([
      "task",
      "owner",
      "due_date",
    ]);
  });
});

describe("SummarySchema", () => {
  const valid = {
    summary: "Team met.",
    decisions: ["Ship v2"],
    action_items: [{ task: "Doc", owner: "Alice", due_date: "2026-04-30" }],
    open_questions: ["Pricing?"],
  };

  it("parses a known-good payload", () => {
    const out = SummarySchema.parse(valid);
    expect(out.action_items[0]?.due_date).toBe("2026-04-30");
  });

  it("normalizes empty-string due_date to undefined", () => {
    const out = SummarySchema.parse({
      ...valid,
      action_items: [{ task: "Doc", owner: "Alice", due_date: "" }],
    });
    expect(out.action_items[0]?.due_date).toBeUndefined();
  });

  it("rejects missing top-level fields", () => {
    expect(() => SummarySchema.parse({ summary: "x" })).toThrow();
  });

  it("rejects an action_item missing owner", () => {
    expect(() =>
      SummarySchema.parse({ ...valid, action_items: [{ task: "Doc", due_date: "" }] }),
    ).toThrow();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm test schema`
Expected: FAIL — module `@/lib/llm/schema` not found.

- [ ] **Step 4: Implement src/lib/llm/schema.ts**

```ts
import { z } from "zod";

export const summaryJsonSchema = {
  type: "object",
  properties: {
    summary: {
      type: "string",
      description: "3–5 sentence executive overview of the meeting.",
    },
    decisions: {
      type: "array",
      description: "Things the meeting settled. Each item is one decision.",
      items: { type: "string" },
    },
    action_items: {
      type: "array",
      description: "Concrete follow-ups from the meeting.",
      items: {
        type: "object",
        properties: {
          task: { type: "string", description: "What needs to be done." },
          owner: {
            type: "string",
            description:
              "Named individual responsible, or 'unassigned' if no owner is clear.",
          },
          due_date: {
            type: "string",
            description:
              "ISO date (YYYY-MM-DD) if stated, empty string otherwise.",
          },
        },
        required: ["task", "owner", "due_date"],
        additionalProperties: false,
      },
    },
    open_questions: {
      type: "array",
      description: "Questions raised in the meeting that were not resolved.",
      items: { type: "string" },
    },
  },
  required: ["summary", "decisions", "action_items", "open_questions"],
  additionalProperties: false,
} as const;

export const ActionItemSchema = z.object({
  task: z.string(),
  owner: z.string(),
  due_date: z.string().transform((s) => (s === "" ? undefined : s)),
});

export const SummarySchema = z.object({
  summary: z.string(),
  decisions: z.array(z.string()),
  action_items: z.array(ActionItemSchema),
  open_questions: z.array(z.string()),
});

export type ActionItem = z.infer<typeof ActionItemSchema>;
export type Summary = z.infer<typeof SummarySchema>;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pnpm test schema`
Expected: 4 passing tests.

- [ ] **Step 6: Commit**

```bash
git add src/lib/llm/schema.ts src/lib/__tests__/schema.test.ts package.json pnpm-lock.yaml
git commit -m "feat(llm): add canonical summary schema (JSON Schema + Zod)"
```

---

## Task 6: Shared types + system prompt

**Files:**
- Create: `src/lib/llm/types.ts`
- Create: `src/lib/llm/prompt.ts`

- [ ] **Step 1: Create src/lib/llm/types.ts**

```ts
import type { Summary } from "./schema";

export type PartialSummary = {
  summary?: string;
  decisions?: string[];
  action_items?: Array<{ task?: string; owner?: string; due_date?: string }>;
  open_questions?: string[];
};

export interface StreamChunk {
  partial: PartialSummary;
}

export interface StreamResult {
  final: Summary;
  usage: { input: number; output: number };
  provider: "claude" | "openai";
  model: string;
}

export interface AdapterArgs {
  transcript: string;
  apiKey: string;
  model?: string;
  signal?: AbortSignal;
}

export interface LLMAdapter {
  name: "claude" | "openai";
  defaultModel: string;
  stream(args: AdapterArgs): AsyncGenerator<StreamChunk, StreamResult, void>;
}
```

- [ ] **Step 2: Create src/lib/llm/prompt.ts**

```ts
export const SYSTEM_PROMPT = `You extract structured summaries from meeting transcripts.

Always call the record_summary tool. Populate every field faithfully from the transcript — do not invent facts.

Field guidance:
- summary: 3–5 sentences describing what happened and why it mattered.
- decisions: each item is one concrete decision the meeting settled.
- action_items: each item has a task, an owner (named individual, or "unassigned" when no owner is clear), and a due_date (ISO YYYY-MM-DD if stated, empty string otherwise).
- open_questions: items raised in the meeting that were not resolved.

If the transcript has no decisions, action items, or open questions, return an empty array for that field. Never omit a field.`;
```

- [ ] **Step 3: Verify**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
git add src/lib/llm/types.ts src/lib/llm/prompt.ts
git commit -m "feat(llm): add adapter types and shared system prompt"
```

---

## Task 7: partial-json parsing test

**Files:**
- Create: `src/lib/__tests__/partial-json.test.ts`

- [ ] **Step 1: Install partial-json**

```bash
pnpm add partial-json@0.1.7
```

- [ ] **Step 2: Write test — src/lib/__tests__/partial-json.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { parse as parsePartial, Allow } from "partial-json";

describe("partial-json", () => {
  it("parses a complete JSON object", () => {
    expect(parsePartial('{"a":1}')).toEqual({ a: 1 });
  });

  it("parses an unterminated string", () => {
    expect(parsePartial('{"a":"he', Allow.ALL)).toEqual({ a: "he" });
  });

  it("parses an unterminated array inside an object", () => {
    expect(parsePartial('{"decisions":["one","two', Allow.ALL)).toEqual({
      decisions: ["one", "two"],
    });
  });

  it("parses progressively as fragments accumulate", () => {
    const frags = ['{"summary":"', "The team ", 'discussed the ', 'plan."}'];
    let buf = "";
    const snapshots: unknown[] = [];
    for (const f of frags) {
      buf += f;
      snapshots.push(parsePartial(buf, Allow.ALL));
    }
    expect(snapshots[0]).toEqual({ summary: "" });
    expect(snapshots[1]).toEqual({ summary: "The team " });
    expect(snapshots[2]).toEqual({ summary: "The team discussed the " });
    expect(snapshots[3]).toEqual({ summary: "The team discussed the plan." });
  });
});
```

- [ ] **Step 3: Run test**

Run: `pnpm test partial-json`
Expected: 4 passing tests.

- [ ] **Step 4: Commit**

```bash
git add src/lib/__tests__/partial-json.test.ts package.json pnpm-lock.yaml
git commit -m "test(llm): pin partial-json streaming behavior with tests"
```

---

## Task 8: Anthropic adapter (TDD with mocked fetch)

**Files:**
- Create: `src/lib/llm/anthropic.ts`
- Create: `src/lib/__tests__/fixtures/golden/product-kickoff.json`
- Create: `src/lib/__tests__/fixtures/streams/anthropic-product-kickoff.txt`
- Create: `src/lib/__tests__/anthropic.test.ts`

- [ ] **Step 1: Install Anthropic SDK (we use fetch directly — SDK is included for type compat)**

```bash
pnpm add @anthropic-ai/sdk@0.30.1
```

- [ ] **Step 2: Create golden fixture src/lib/__tests__/fixtures/golden/product-kickoff.json**

```json
{
  "summary": "The product, design, and engineering leads aligned on the Q2 launch of the collaborative editing feature. They agreed to cut scope on presence indicators to protect the June 15 ship date, and assigned migration and QA ownership. One open question remains around pricing for enterprise tiers.",
  "decisions": [
    "Ship collaborative editing by June 15 without presence indicators in v1.",
    "Freeze the feature surface on May 30 to allow two weeks of QA.",
    "Use the existing CRDT library rather than rolling a custom sync engine."
  ],
  "action_items": [
    { "task": "Write the data-model migration plan and share for review.", "owner": "Alice", "due_date": "2026-04-30" },
    { "task": "Set up the QA environment and recruit three design-partner testers.", "owner": "Marcus", "due_date": "2026-05-15" },
    { "task": "Draft a one-pager on the decision to defer presence indicators.", "owner": "Priya", "due_date": "" }
  ],
  "open_questions": [
    "How should we price collaborative editing for enterprise customers?"
  ]
}
```

- [ ] **Step 3: Create stream fixture src/lib/__tests__/fixtures/streams/anthropic-product-kickoff.txt**

Contents (the test substitutes `__GOLDEN_JSON__` with the escaped golden JSON at runtime):

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_test","type":"message","role":"assistant","model":"claude-sonnet-4-6","content":[],"stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":1200,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"tool_use","id":"toolu_test","name":"record_summary","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"__GOLDEN_JSON__"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use","stop_sequence":null},"usage":{"output_tokens":450}}

event: message_stop
data: {"type":"message_stop"}

```

- [ ] **Step 4: Write failing test — src/lib/__tests__/anthropic.test.ts**

```ts
import { describe, it, expect, beforeEach, afterEach, vi } from "vitest";
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { AnthropicAdapter } from "@/lib/llm/anthropic";

const golden = JSON.parse(
  readFileSync(join(__dirname, "fixtures/golden/product-kickoff.json"), "utf8"),
) as Record<string, unknown>;

// Escape the golden for embedding inside a JSON string value (which is itself
// already inside an SSE data: line).
const escapedGolden = JSON.stringify(JSON.stringify(golden)).slice(1, -1);

const rawStream = readFileSync(
  join(__dirname, "fixtures/streams/anthropic-product-kickoff.txt"),
  "utf8",
).replace("__GOLDEN_JSON__", escapedGolden);

function makeSseResponse(body: string): Response {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    start(controller) {
      controller.enqueue(encoder.encode(body));
      controller.close();
    },
  });
  return new Response(stream, {
    status: 200,
    headers: { "content-type": "text/event-stream" },
  });
}

describe("AnthropicAdapter", () => {
  let fetchSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    fetchSpy = vi.spyOn(globalThis, "fetch").mockResolvedValue(makeSseResponse(rawStream));
  });

  afterEach(() => {
    fetchSpy.mockRestore();
  });

  it("streams partial chunks and returns validated final summary", async () => {
    const adapter = new AnthropicAdapter();
    const chunks: unknown[] = [];
    const gen = adapter.stream({
      transcript: "Product kickoff transcript...",
      apiKey: "sk-ant-test",
    });

    let result: Awaited<ReturnType<typeof gen.next>> | null = null;
    while (true) {
      const next = await gen.next();
      if (next.done) {
        result = next;
        break;
      }
      chunks.push(next.value.partial);
    }

    expect(chunks.length).toBeGreaterThan(0);
    expect(result?.value.final.summary).toEqual(golden.summary);
    expect(result?.value.final.action_items[0]?.due_date).toBe("2026-04-30");
    // Empty-string due_date normalized to undefined by Zod
    expect(result?.value.final.action_items[2]?.due_date).toBeUndefined();
    expect(result?.value.provider).toBe("claude");
    expect(result?.value.usage.output).toBe(450);
    expect(result?.value.usage.input).toBe(1200);
  });

  it("sends tool_choice forcing record_summary", async () => {
    const adapter = new AnthropicAdapter();
    const gen = adapter.stream({ transcript: "x", apiKey: "k" });
    while (!(await gen.next()).done) {
      /* drain */
    }

    expect(fetchSpy).toHaveBeenCalledOnce();
    const [, init] = fetchSpy.mock.calls[0]!;
    const body = JSON.parse((init as RequestInit).body as string);
    expect(body.tool_choice).toEqual({ type: "tool", name: "record_summary" });
    expect(body.tools[0].name).toBe("record_summary");
    expect(body.tools[0].input_schema.required).toContain("action_items");
    expect(body.stream).toBe(true);
  });
});
```

- [ ] **Step 5: Run test to verify it fails**

Run: `pnpm test anthropic`
Expected: FAIL — module `@/lib/llm/anthropic` not found.

- [ ] **Step 6: Implement src/lib/llm/anthropic.ts**

```ts
import { parse as parsePartial, Allow } from "partial-json";
import { SYSTEM_PROMPT } from "./prompt";
import { SummarySchema, summaryJsonSchema, type Summary } from "./schema";
import type { AdapterArgs, LLMAdapter, StreamChunk, StreamResult } from "./types";

const DEFAULT_MODEL = "claude-sonnet-4-6";
const API_URL = "https://api.anthropic.com/v1/messages";

export class AnthropicAdapter implements LLMAdapter {
  readonly name = "claude" as const;
  readonly defaultModel = DEFAULT_MODEL;

  async *stream(args: AdapterArgs): AsyncGenerator<StreamChunk, StreamResult, void> {
    const model = args.model ?? process.env.ANTHROPIC_MODEL ?? DEFAULT_MODEL;

    const res = await fetch(API_URL, {
      method: "POST",
      signal: args.signal,
      headers: {
        "content-type": "application/json",
        "x-api-key": args.apiKey,
        "anthropic-version": "2023-06-01",
      },
      body: JSON.stringify({
        model,
        max_tokens: 2048,
        stream: true,
        system: SYSTEM_PROMPT,
        tools: [
          {
            name: "record_summary",
            description: "Record a structured summary of the meeting transcript.",
            input_schema: summaryJsonSchema,
          },
        ],
        tool_choice: { type: "tool", name: "record_summary" },
        messages: [{ role: "user", content: args.transcript }],
      }),
    });

    if (!res.ok || !res.body) {
      const text = await res.text().catch(() => "");
      throw new AdapterError(mapAnthropicStatus(res.status), `Anthropic ${res.status}: ${text}`);
    }

    let buffer = "";
    let outputTokens = 0;
    let inputTokens = 0;

    for await (const event of parseSseStream(res.body)) {
      let payload: unknown;
      try {
        payload = JSON.parse(event.data);
      } catch {
        continue;
      }

      if (!isObj(payload)) continue;

      if (payload.type === "message_start" && isObj(payload.message) && isObj(payload.message.usage)) {
        inputTokens = Number(payload.message.usage.input_tokens) || 0;
      }

      if (
        payload.type === "content_block_delta" &&
        isObj(payload.delta) &&
        payload.delta.type === "input_json_delta" &&
        typeof payload.delta.partial_json === "string"
      ) {
        buffer += payload.delta.partial_json;
        const partial = parseBuffer(buffer);
        if (partial) yield { partial };
      }

      if (payload.type === "message_delta" && isObj(payload.usage)) {
        outputTokens = Number(payload.usage.output_tokens) || outputTokens;
      }
    }

    const final = SummarySchema.parse(parseBuffer(buffer) ?? {});
    return {
      final,
      usage: { input: inputTokens, output: outputTokens },
      provider: this.name,
      model,
    } satisfies StreamResult;
  }
}

function parseBuffer(buf: string): Summary | null {
  if (!buf) return null;
  try {
    return parsePartial(buf, Allow.ALL) as Summary;
  } catch {
    return null;
  }
}

function isObj(v: unknown): v is Record<string, unknown> {
  return typeof v === "object" && v !== null;
}

export class AdapterError extends Error {
  constructor(
    public code: "invalid_key" | "rate_limited" | "provider_unavailable" | "schema_mismatch",
    message: string,
  ) {
    super(message);
  }
}

function mapAnthropicStatus(status: number): AdapterError["code"] {
  if (status === 401) return "invalid_key";
  if (status === 429) return "rate_limited";
  return "provider_unavailable";
}

async function* parseSseStream(body: ReadableStream<Uint8Array>): AsyncGenerator<{
  event: string;
  data: string;
}> {
  const reader = body.getReader();
  const decoder = new TextDecoder();
  let buf = "";
  let currentEvent = "";
  let currentData = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });

    let idx: number;
    while ((idx = buf.indexOf("\n")) !== -1) {
      const line = buf.slice(0, idx).replace(/\r$/, "");
      buf = buf.slice(idx + 1);
      if (line === "") {
        if (currentEvent || currentData) {
          yield { event: currentEvent, data: currentData };
          currentEvent = "";
          currentData = "";
        }
        continue;
      }
      if (line.startsWith("event:")) currentEvent = line.slice(6).trim();
      else if (line.startsWith("data:")) currentData += line.slice(5).trim();
    }
  }
}
```

- [ ] **Step 7: Run test**

Run: `pnpm test anthropic`
Expected: 2 passing tests.

- [ ] **Step 8: Commit**

```bash
git add src/lib/llm/anthropic.ts src/lib/__tests__/anthropic.test.ts src/lib/__tests__/fixtures package.json pnpm-lock.yaml
git commit -m "feat(llm): Anthropic adapter with streaming tool-call extraction"
```

---

## Task 9: OpenAI adapter (TDD with mocked fetch)

**Files:**
- Create: `src/lib/llm/openai.ts`
- Create: `src/lib/__tests__/fixtures/streams/openai-product-kickoff.txt`
- Create: `src/lib/__tests__/openai.test.ts`

- [ ] **Step 1: Install OpenAI SDK (types; we call via fetch)**

```bash
pnpm add openai@4.73.1
```

- [ ] **Step 2: Create stream fixture src/lib/__tests__/fixtures/streams/openai-product-kickoff.txt**

```
data: {"id":"c1","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":""}}]}

data: {"id":"c1","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"__GOLDEN_JSON__"}}]}

data: {"id":"c1","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: {"id":"c1","object":"chat.completion.chunk","choices":[],"usage":{"prompt_tokens":1100,"completion_tokens":420,"total_tokens":1520}}

data: [DONE]

```

- [ ] **Step 3: Write failing test — src/lib/__tests__/openai.test.ts**

```ts
import { describe, it, expect, beforeEach, afterEach, vi } from "vitest";
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { OpenAIAdapter } from "@/lib/llm/openai";

const golden = JSON.parse(
  readFileSync(join(__dirname, "fixtures/golden/product-kickoff.json"), "utf8"),
) as Record<string, unknown>;

const escapedGolden = JSON.stringify(JSON.stringify(golden)).slice(1, -1);

const rawStream = readFileSync(
  join(__dirname, "fixtures/streams/openai-product-kickoff.txt"),
  "utf8",
).replace("__GOLDEN_JSON__", escapedGolden);

function makeStream(body: string): Response {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    start(controller) {
      controller.enqueue(encoder.encode(body));
      controller.close();
    },
  });
  return new Response(stream, { status: 200, headers: { "content-type": "text/event-stream" } });
}

describe("OpenAIAdapter", () => {
  let fetchSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    fetchSpy = vi.spyOn(globalThis, "fetch").mockResolvedValue(makeStream(rawStream));
  });
  afterEach(() => fetchSpy.mockRestore());

  it("streams partial chunks and returns validated final summary", async () => {
    const adapter = new OpenAIAdapter();
    const gen = adapter.stream({ transcript: "t", apiKey: "sk-test" });
    const chunks: unknown[] = [];
    // biome-ignore lint/style/useConst: we reassign in the loop
    let next = await gen.next();
    while (!next.done) {
      chunks.push(next.value.partial);
      next = await gen.next();
    }

    expect(chunks.length).toBeGreaterThan(0);
    expect(next.value.final.summary).toEqual(golden.summary);
    expect(next.value.provider).toBe("openai");
    expect(next.value.usage.output).toBe(420);
    expect(next.value.usage.input).toBe(1100);
  });

  it("sends response_format json_schema with strict: true", async () => {
    const adapter = new OpenAIAdapter();
    const gen = adapter.stream({ transcript: "t", apiKey: "sk-test" });
    while (!(await gen.next()).done) {
      /* drain */
    }
    const [, init] = fetchSpy.mock.calls[0]!;
    const body = JSON.parse((init as RequestInit).body as string);
    expect(body.response_format.type).toBe("json_schema");
    expect(body.response_format.json_schema.strict).toBe(true);
    expect(body.response_format.json_schema.name).toBe("record_summary");
    expect(body.stream).toBe(true);
    expect(body.stream_options.include_usage).toBe(true);
  });
});
```

- [ ] **Step 4: Run test to verify failure**

Run: `pnpm test openai`
Expected: FAIL — module not found.

- [ ] **Step 5: Implement src/lib/llm/openai.ts**

```ts
import { parse as parsePartial, Allow } from "partial-json";
import { SYSTEM_PROMPT } from "./prompt";
import { SummarySchema, summaryJsonSchema, type Summary } from "./schema";
import { AdapterError } from "./anthropic";
import type { AdapterArgs, LLMAdapter, StreamChunk, StreamResult } from "./types";

const DEFAULT_MODEL = "gpt-4o-2024-08-06";
const API_URL = "https://api.openai.com/v1/chat/completions";

export class OpenAIAdapter implements LLMAdapter {
  readonly name = "openai" as const;
  readonly defaultModel = DEFAULT_MODEL;

  async *stream(args: AdapterArgs): AsyncGenerator<StreamChunk, StreamResult, void> {
    const model = args.model ?? process.env.OPENAI_MODEL ?? DEFAULT_MODEL;

    const res = await fetch(API_URL, {
      method: "POST",
      signal: args.signal,
      headers: {
        "content-type": "application/json",
        authorization: `Bearer ${args.apiKey}`,
      },
      body: JSON.stringify({
        model,
        stream: true,
        stream_options: { include_usage: true },
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "record_summary",
            strict: true,
            schema: summaryJsonSchema,
          },
        },
        messages: [
          { role: "system", content: SYSTEM_PROMPT },
          { role: "user", content: args.transcript },
        ],
      }),
    });

    if (!res.ok || !res.body) {
      const text = await res.text().catch(() => "");
      throw new AdapterError(mapOpenAIStatus(res.status), `OpenAI ${res.status}: ${text}`);
    }

    let buffer = "";
    let outputTokens = 0;
    let inputTokens = 0;

    for await (const event of parseSseData(res.body)) {
      if (event === "[DONE]") break;

      let payload: unknown;
      try {
        payload = JSON.parse(event);
      } catch {
        continue;
      }

      if (!isObj(payload)) continue;

      const choice = Array.isArray(payload.choices) ? payload.choices[0] : undefined;
      if (isObj(choice) && isObj(choice.delta) && typeof choice.delta.content === "string") {
        buffer += choice.delta.content;
        const partial = parseBuffer(buffer);
        if (partial) yield { partial };
      }

      if (isObj(payload.usage)) {
        inputTokens = Number(payload.usage.prompt_tokens) || inputTokens;
        outputTokens = Number(payload.usage.completion_tokens) || outputTokens;
      }
    }

    const final = SummarySchema.parse(parseBuffer(buffer) ?? {});
    return {
      final,
      usage: { input: inputTokens, output: outputTokens },
      provider: this.name,
      model,
    } satisfies StreamResult;
  }
}

function parseBuffer(buf: string): Summary | null {
  if (!buf) return null;
  try {
    return parsePartial(buf, Allow.ALL) as Summary;
  } catch {
    return null;
  }
}

function isObj(v: unknown): v is Record<string, unknown> {
  return typeof v === "object" && v !== null;
}

function mapOpenAIStatus(status: number): AdapterError["code"] {
  if (status === 401) return "invalid_key";
  if (status === 429) return "rate_limited";
  return "provider_unavailable";
}

async function* parseSseData(body: ReadableStream<Uint8Array>): AsyncGenerator<string> {
  const reader = body.getReader();
  const decoder = new TextDecoder();
  let buf = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });

    let idx: number;
    while ((idx = buf.indexOf("\n")) !== -1) {
      const line = buf.slice(0, idx).replace(/\r$/, "");
      buf = buf.slice(idx + 1);
      if (line.startsWith("data:")) yield line.slice(5).trim();
    }
  }
}
```

- [ ] **Step 6: Run test**

Run: `pnpm test openai`
Expected: 2 passing tests.

- [ ] **Step 7: Commit**

```bash
git add src/lib/llm/openai.ts src/lib/__tests__/openai.test.ts src/lib/__tests__/fixtures/streams/openai-product-kickoff.txt package.json pnpm-lock.yaml
git commit -m "feat(llm): OpenAI adapter with streaming strict json_schema"
```

---

## Task 10: Adapter picker

**Files:**
- Create: `src/lib/llm/index.ts`
- Create: `src/lib/__tests__/pick-adapter.test.ts`

- [ ] **Step 1: Write failing test — src/lib/__tests__/pick-adapter.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { pickAdapter, resolveProviderKey } from "@/lib/llm";

describe("pickAdapter", () => {
  it("returns Anthropic for claude-* model ids", () => {
    expect(pickAdapter("claude-sonnet-4-6").name).toBe("claude");
    expect(pickAdapter("claude-opus-4-7").name).toBe("claude");
  });

  it("returns OpenAI for gpt-* model ids", () => {
    expect(pickAdapter("gpt-4o").name).toBe("openai");
    expect(pickAdapter("gpt-4o-2024-08-06").name).toBe("openai");
  });

  it("throws on unknown model ids", () => {
    expect(() => pickAdapter("bard")).toThrow(/unknown model/i);
  });
});

describe("resolveProviderKey", () => {
  it("returns BYO-key when provided", () => {
    expect(resolveProviderKey("claude", "sk-user", { ANTHROPIC_API_KEY: "sk-server" })).toBe(
      "sk-user",
    );
  });

  it("falls back to server env for each provider", () => {
    expect(resolveProviderKey("claude", undefined, { ANTHROPIC_API_KEY: "sk-a" })).toBe("sk-a");
    expect(resolveProviderKey("openai", undefined, { OPENAI_API_KEY: "sk-o" })).toBe("sk-o");
  });

  it("throws when neither BYO-key nor server env is available", () => {
    expect(() => resolveProviderKey("claude", undefined, {})).toThrow(/no api key/i);
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `pnpm test pick-adapter`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement src/lib/llm/index.ts**

```ts
import { AnthropicAdapter } from "./anthropic";
import { OpenAIAdapter } from "./openai";
import type { LLMAdapter } from "./types";

export const DEFAULT_MODEL = "claude-sonnet-4-6" as const;

export function pickAdapter(model: string): LLMAdapter {
  if (model.startsWith("claude-")) return new AnthropicAdapter();
  if (model.startsWith("gpt-")) return new OpenAIAdapter();
  throw new Error(`unknown model: ${model}`);
}

export function resolveProviderKey(
  provider: "claude" | "openai",
  byoKey: string | undefined,
  env: Record<string, string | undefined>,
): string {
  if (byoKey && byoKey.trim()) return byoKey.trim();
  const serverKey = provider === "claude" ? env.ANTHROPIC_API_KEY : env.OPENAI_API_KEY;
  if (serverKey && serverKey.trim()) return serverKey.trim();
  throw new Error(`no API key available for ${provider}`);
}

export { AnthropicAdapter, OpenAIAdapter };
export type { LLMAdapter } from "./types";
```

- [ ] **Step 4: Run test**

Run: `pnpm test pick-adapter`
Expected: 6 passing tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/llm/index.ts src/lib/__tests__/pick-adapter.test.ts
git commit -m "feat(llm): provider picker and key resolver"
```

---

## Task 11: Request body validator

**Files:**
- Create: `src/lib/validate.ts`
- Create: `src/lib/__tests__/validate.test.ts`

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect } from "vitest";
import { SummarizeRequestSchema, MAX_TRANSCRIPT_CHARS } from "@/lib/validate";

describe("SummarizeRequestSchema", () => {
  it("accepts a minimal body and defaults model to claude-sonnet-4-6", () => {
    const out = SummarizeRequestSchema.parse({ transcript: "hello" });
    expect(out.model).toBe("claude-sonnet-4-6");
    expect(out.byoKey).toBeUndefined();
  });

  it("accepts an explicit model and byoKey", () => {
    const out = SummarizeRequestSchema.parse({
      transcript: "x",
      model: "gpt-4o",
      byoKey: "sk-1",
    });
    expect(out.model).toBe("gpt-4o");
    expect(out.byoKey).toBe("sk-1");
  });

  it("rejects empty transcripts", () => {
    expect(() => SummarizeRequestSchema.parse({ transcript: "" })).toThrow();
  });

  it("rejects transcripts over the max length", () => {
    expect(() =>
      SummarizeRequestSchema.parse({ transcript: "a".repeat(MAX_TRANSCRIPT_CHARS + 1) }),
    ).toThrow();
  });

  it("rejects unknown models", () => {
    expect(() => SummarizeRequestSchema.parse({ transcript: "x", model: "bard" })).toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `pnpm test validate`
Expected: FAIL.

- [ ] **Step 3: Implement src/lib/validate.ts**

```ts
import { z } from "zod";

export const MAX_TRANSCRIPT_CHARS = 50_000;

export const SummarizeRequestSchema = z.object({
  transcript: z.string().min(1).max(MAX_TRANSCRIPT_CHARS),
  model: z
    .enum(["claude-sonnet-4-6", "gpt-4o", "gpt-4o-2024-08-06"])
    .default("claude-sonnet-4-6"),
  byoKey: z.string().trim().min(1).optional(),
});

export type SummarizeRequest = z.infer<typeof SummarizeRequestSchema>;
```

- [ ] **Step 4: Run test**

Run: `pnpm test validate`
Expected: 5 passing tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/validate.ts src/lib/__tests__/validate.test.ts
git commit -m "feat: request body validator for /api/summarize"
```

---

## Task 12: Rate limiter

**Files:**
- Create: `src/lib/rate-limit.ts`

No test — thin wrapper around a well-tested library; the no-op fallback is trivial.

- [ ] **Step 1: Install Upstash deps**

```bash
pnpm add @upstash/ratelimit@2.0.4 @upstash/redis@1.34.3
```

- [ ] **Step 2: Create src/lib/rate-limit.ts**

```ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

type LimitResult = { success: true } | { success: false; reset: number };

let limiter: Ratelimit | null = null;
let warnedMissing = false;

function getLimiter(): Ratelimit | null {
  if (limiter) return limiter;

  const url = process.env.UPSTASH_REDIS_REST_URL;
  const token = process.env.UPSTASH_REDIS_REST_TOKEN;

  if (!url || !token) {
    if (!warnedMissing) {
      console.warn(
        "[rate-limit] UPSTASH_REDIS_REST_URL/TOKEN not set — rate limiting disabled.",
      );
      warnedMissing = true;
    }
    return null;
  }

  limiter = new Ratelimit({
    redis: new Redis({ url, token }),
    limiter: Ratelimit.fixedWindow(10, "24 h"),
    analytics: false,
    prefix: "meetingscribe",
  });
  return limiter;
}

export async function checkRateLimit(ip: string): Promise<LimitResult> {
  const rl = getLimiter();
  if (!rl) return { success: true };
  const res = await rl.limit(ip);
  if (res.success) return { success: true };
  return { success: false, reset: res.reset };
}
```

- [ ] **Step 3: Verify typecheck**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
git add src/lib/rate-limit.ts package.json pnpm-lock.yaml
git commit -m "feat: Upstash rate limiter with graceful no-op fallback"
```

---

## Task 13: /api/summarize route (Edge, SSE)

**Files:**
- Create: `src/app/api/summarize/route.ts`

- [ ] **Step 1: Create src/app/api/summarize/route.ts**

```ts
import { NextResponse } from "next/server";
import { pickAdapter, resolveProviderKey } from "@/lib/llm";
import { AdapterError } from "@/lib/llm/anthropic";
import { SummarizeRequestSchema } from "@/lib/validate";
import { checkRateLimit } from "@/lib/rate-limit";

export const runtime = "edge";

export async function POST(req: Request) {
  let body: unknown;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ message: "Invalid JSON body" }, { status: 400 });
  }

  const parsed = SummarizeRequestSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { message: parsed.error.issues[0]?.message ?? "Invalid request" },
      { status: 400 },
    );
  }

  const { transcript, model, byoKey } = parsed.data;

  if (!byoKey) {
    const ip = req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() || "unknown";
    const rl = await checkRateLimit(ip);
    if (!rl.success) {
      return NextResponse.json(
        {
          message: "Rate limit reached. Add your own API key to continue.",
          code: "rate_limited",
        },
        { status: 429 },
      );
    }
  }

  let adapter;
  let apiKey;
  try {
    adapter = pickAdapter(model);
    apiKey = resolveProviderKey(adapter.name, byoKey, process.env as Record<string, string | undefined>);
  } catch (e) {
    return NextResponse.json(
      { message: e instanceof Error ? e.message : "Setup error" },
      { status: 400 },
    );
  }

  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();
      const write = (event: string, data: unknown) => {
        controller.enqueue(encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`));
      };

      try {
        const gen = adapter.stream({ transcript, apiKey, model, signal: req.signal });
        while (true) {
          const next = await gen.next();
          if (next.done) {
            write("done", {
              ...next.value.final,
              _meta: {
                provider: next.value.provider,
                model: next.value.model,
                usage: next.value.usage,
              },
            });
            break;
          }
          write("partial", next.value.partial);
        }
      } catch (e) {
        if (e instanceof AdapterError) {
          write("error", { message: e.message, code: e.code });
        } else if (e instanceof Error && e.name === "AbortError") {
          // client went away — don't write
        } else {
          write("error", {
            message: e instanceof Error ? e.message : "Unknown error",
            code: "provider_unavailable",
          });
        }
      } finally {
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      "content-type": "text/event-stream; charset=utf-8",
      "cache-control": "no-cache, no-transform",
      connection: "keep-alive",
    },
  });
}
```

- [ ] **Step 2: Verify typecheck and build**

Run: `pnpm typecheck && pnpm build`
Expected: passes.

- [ ] **Step 3: Commit**

```bash
git add src/app/api/summarize/route.ts
git commit -m "feat(api): Edge /api/summarize with SSE streaming"
```

---

## Task 14: Export helper (TDD)

**Files:**
- Create: `src/lib/export.ts`
- Create: `src/lib/__tests__/export.test.ts`

- [ ] **Step 1: Write failing test**

```ts
import { describe, it, expect } from "vitest";
import { toMarkdown } from "@/lib/export";
import type { Summary } from "@/lib/llm/schema";

const summary: Summary = {
  summary: "We shipped the thing.",
  decisions: ["Ship Friday", "Deprecate v1"],
  action_items: [
    { task: "Write docs", owner: "Alice", due_date: "2026-05-01" },
    { task: "File follow-up", owner: "unassigned", due_date: undefined },
  ],
  open_questions: ["How do we price v2?"],
};

describe("toMarkdown", () => {
  it("renders all four sections when populated", () => {
    const md = toMarkdown(summary);
    expect(md).toContain("## Summary\n\nWe shipped the thing.");
    expect(md).toContain("## Decisions\n\n- Ship Friday\n- Deprecate v1");
    expect(md).toContain("**Alice** · due 2026-05-01");
    expect(md).toContain("**unassigned**");
    expect(md).not.toContain("due undefined");
    expect(md).toContain("## Open Questions\n\n- How do we price v2?");
  });

  it("omits sections that are empty", () => {
    const md = toMarkdown({
      ...summary,
      decisions: [],
      open_questions: [],
    });
    expect(md).not.toContain("## Decisions");
    expect(md).not.toContain("## Open Questions");
  });
});
```

- [ ] **Step 2: Run test to verify failure**

Run: `pnpm test export`
Expected: FAIL.

- [ ] **Step 3: Implement src/lib/export.ts**

```ts
import type { Summary } from "@/lib/llm/schema";

export function toMarkdown(s: Summary): string {
  const parts: string[] = [];
  parts.push(`## Summary\n\n${s.summary}`);

  if (s.decisions.length > 0) {
    parts.push(`## Decisions\n\n${s.decisions.map((d) => `- ${d}`).join("\n")}`);
  }

  if (s.action_items.length > 0) {
    parts.push(
      `## Action Items\n\n${s.action_items
        .map((a) => `- ${a.task} — **${a.owner}**${a.due_date ? ` · due ${a.due_date}` : ""}`)
        .join("\n")}`,
    );
  }

  if (s.open_questions.length > 0) {
    parts.push(`## Open Questions\n\n${s.open_questions.map((q) => `- ${q}`).join("\n")}`);
  }

  return parts.join("\n\n");
}
```

- [ ] **Step 4: Run test**

Run: `pnpm test export`
Expected: 2 passing tests.

- [ ] **Step 5: Commit**

```bash
git add src/lib/export.ts src/lib/__tests__/export.test.ts
git commit -m "feat: Markdown export for summaries"
```

---

## Task 15: useSummaryStream hook

**Files:**
- Create: `src/lib/hooks/useSummaryStream.ts`

- [ ] **Step 1: Create src/lib/hooks/useSummaryStream.ts**

```ts
"use client";
import { useCallback, useRef, useState } from "react";
import type { PartialSummary } from "@/lib/llm/types";
import type { Summary } from "@/lib/llm/schema";

export type StreamStatus = "idle" | "streaming" | "done" | "error";
export type StreamError = { message: string; code?: string };
type FinalWithMeta = Summary & {
  _meta?: { provider: string; model: string; usage: { input: number; output: number } };
};

export interface UseSummaryStream {
  partial: PartialSummary;
  final: FinalWithMeta | null;
  error: StreamError | null;
  status: StreamStatus;
  start: (args: { transcript: string; model: string; byoKey?: string }) => void;
  abort: () => void;
}

export function useSummaryStream(): UseSummaryStream {
  const [partial, setPartial] = useState<PartialSummary>({});
  const [final, setFinal] = useState<FinalWithMeta | null>(null);
  const [error, setError] = useState<StreamError | null>(null);
  const [status, setStatus] = useState<StreamStatus>("idle");
  const controllerRef = useRef<AbortController | null>(null);

  const abort = useCallback(() => {
    controllerRef.current?.abort();
  }, []);

  const start = useCallback(
    async ({
      transcript,
      model,
      byoKey,
    }: {
      transcript: string;
      model: string;
      byoKey?: string;
    }) => {
      controllerRef.current?.abort();
      const controller = new AbortController();
      controllerRef.current = controller;

      setPartial({});
      setFinal(null);
      setError(null);
      setStatus("streaming");

      const dispatch = (event: string, data: string) => {
        try {
          const payload = JSON.parse(data);
          if (event === "partial") setPartial(payload);
          else if (event === "done") {
            setFinal(payload);
            setStatus("done");
          } else if (event === "error") {
            setError({ message: payload.message, code: payload.code });
            setStatus("error");
          }
        } catch {
          /* ignore malformed */
        }
      };

      try {
        const res = await fetch("/api/summarize", {
          method: "POST",
          headers: { "content-type": "application/json" },
          body: JSON.stringify({ transcript, model, byoKey }),
          signal: controller.signal,
        });

        if (!res.ok) {
          const body = await res.json().catch(() => ({ message: `HTTP ${res.status}` }));
          setError({ message: body.message ?? `HTTP ${res.status}`, code: body.code });
          setStatus("error");
          return;
        }
        if (!res.body) {
          setError({ message: "No response body" });
          setStatus("error");
          return;
        }

        const reader = res.body.getReader();
        const decoder = new TextDecoder();
        let buf = "";
        let currentEvent = "";
        let currentData = "";

        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          buf += decoder.decode(value, { stream: true });
          let idx: number;
          while ((idx = buf.indexOf("\n")) !== -1) {
            const line = buf.slice(0, idx).replace(/\r$/, "");
            buf = buf.slice(idx + 1);
            if (line === "") {
              if (currentEvent) dispatch(currentEvent, currentData);
              currentEvent = "";
              currentData = "";
              continue;
            }
            if (line.startsWith("event:")) currentEvent = line.slice(6).trim();
            else if (line.startsWith("data:")) currentData += line.slice(5).trim();
          }
        }
      } catch (e) {
        if ((e as Error)?.name === "AbortError") return;
        setError({ message: e instanceof Error ? e.message : "Network error" });
        setStatus("error");
      }
    },
    [],
  );

  return { partial, final, error, status, start, abort };
}
```

- [ ] **Step 2: Verify typecheck**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 3: Commit**

```bash
git add src/lib/hooks/useSummaryStream.ts
git commit -m "feat: useSummaryStream hook consumes SSE from /api/summarize"
```

---

## Task 16: Sample transcripts + golden fixtures

**Files:**
- Create: `src/lib/samples/product-kickoff.ts`
- Create: `src/lib/samples/eng-standup.ts`
- Create: `src/lib/samples/customer-call.ts`
- Create: `src/lib/samples/index.ts`
- Create: `src/lib/__tests__/fixtures/golden/eng-standup.json`
- Create: `src/lib/__tests__/fixtures/golden/customer-call.json`

- [ ] **Step 1: Create src/lib/samples/product-kickoff.ts**

```ts
export const productKickoff = `
[2026-04-15 Product kickoff — Q2 collaborative editing]

Dana (PM): Thanks everyone. Goal today: commit on scope for the Q2 launch of collaborative editing. We said June 15. Is that still real?

Marcus (Eng lead): Real if we cut presence indicators from v1. With presence in, I need another three weeks minimum.

Alice (Infra): I agree. The data-model migration alone is non-trivial — I haven't even started the plan.

Priya (Design): Presence is the flashy part. If we cut it, what's the story?

Dana: The story is that collaborative editing lands June 15 without presence. Presence ships in a follow-up. That lets us start real customer testing two months earlier.

Marcus: Works for me. Alice, can you own the migration plan? Share by end of month.

Alice: Yes. April 30.

Marcus: I'll own the QA environment and recruit three design partners. Call it May 15.

Priya: I'll write a one-pager explaining why we're deferring presence. No hard date — it's internal comms, can float.

Dana: Also: we need to freeze the feature surface on May 30 so QA has two weeks.

Marcus: Agreed. Write that down as a decision.

Dana: And on the sync engine — we're using the existing CRDT library, not rolling our own.

Marcus: Confirmed. Rolling our own was a bad idea.

Priya: One thing unresolved — how do we price this for enterprise? The PLG motion suggests free. Finance wants an add-on SKU. We didn't settle it.

Dana: Right, open question. Let's pick that up with finance next week.
`.trim();
```

- [ ] **Step 2: Create src/lib/samples/eng-standup.ts**

```ts
export const engStandup = `
[2026-04-22 Platform standup]

Raj: Standup, three blockers.

Raj: Yesterday — finished the gRPC migration for the billing service. Today — starting on webhook retries. Blocker: I need a decision on whether we use Inngest or roll our own retry queue. Cost difference is meaningful.

Sam: Yesterday — paged at 3am for the ingest lag incident. Root cause: a consumer was silently dying on malformed messages and not restarting. Fix is in, postmortem in the channel. Today — writing the retro doc. Blocker: I need someone to pair on adding the dead-letter queue alert by Friday.

Nina: Yesterday — finished performance audit for the search page. P99 went from 2.4s to 340ms. Today — same for the settings page. No blockers.

Raj: On the Inngest decision — can we make a call today? I need to move.

Sam: I've used Inngest. For our volume it's a clear win. Let's just do that.

Raj: OK, calling it — Inngest. Decision.

Sam: Dead-letter alert?

Nina: I'll pair on it. By Friday, yes.

Raj: Sam's retro doc — who's the audience?

Sam: Eng-wide. I'll send by Monday.

Raj: One thing I want to flag — we still haven't decided how we want to handle backfills for the new billing schema. Not urgent but we should pick it up.
`.trim();
```

- [ ] **Step 3: Create src/lib/samples/customer-call.ts**

```ts
export const customerCall = `
[2026-04-18 Discovery call — Acme Corp]

Sarah (AE): Thanks for making time, Jordan. I know you run ops at Acme. What prompted the outreach?

Jordan (Acme Ops): We're drowning in meeting notes. Every cross-functional sync becomes a Notion page, and nothing is connected to our downstream systems — Linear, Salesforce, Slack. Nobody actions anything.

Sarah: How big is the team?

Jordan: Forty-two people across product, CS, and support.

Sarah: And what have you tried?

Jordan: Otter. People didn't adopt it. The summaries were generic and the export was a blob of text.

Sarah: Got it. Our angle is structured output — every summary is JSON your tools can consume. Linear tickets, Slack messages, whatever.

Jordan: Can you send pricing? I'd want to pilot with maybe 10 people first.

Sarah: I can send pricing today. We do a 30-day pilot at no cost for teams under 20.

Jordan: What's the security posture? We can't send customer conversations to a third party without a DPA.

Sarah: We have a DPA. I'll include it with the pilot agreement. If you need self-host, we support Docker Compose — there's a roadmap item for air-gapped.

Jordan: Self-host is nice to have, not a hard requirement yet.

Sarah: Next step: I send pilot paperwork and pricing by tomorrow. You loop in your sec team for the DPA review.

Jordan: Works. One thing I'm unsure about — we'd want a Linear integration specifically. Is that live?

Sarah: Not yet. Roadmap Q3.

Jordan: Noted. Open item for us: does a Zapier bridge work in the interim?
`.trim();
```

- [ ] **Step 4: Create src/lib/samples/index.ts**

```ts
import { productKickoff } from "./product-kickoff";
import { engStandup } from "./eng-standup";
import { customerCall } from "./customer-call";

export const samples = [
  { id: "product-kickoff", label: "Product kickoff", transcript: productKickoff },
  { id: "eng-standup", label: "Eng standup", transcript: engStandup },
  { id: "customer-call", label: "Customer discovery", transcript: customerCall },
] as const;

export type SampleId = (typeof samples)[number]["id"];
```

- [ ] **Step 5: Create src/lib/__tests__/fixtures/golden/eng-standup.json**

```json
{
  "summary": "The platform team ran through blockers at standup. Raj made a call to adopt Inngest for webhook retries. Sam surfaced an ingest-lag incident, committed to a retro, and lined up a dead-letter queue alert for Friday. Nina reported a significant performance win on the search page. A question remains around billing-schema backfills.",
  "decisions": [
    "Use Inngest for webhook retries rather than rolling a custom retry queue.",
    "Pair on adding a dead-letter queue alert for the ingest pipeline by Friday."
  ],
  "action_items": [
    { "task": "Start implementing webhook retries using Inngest.", "owner": "Raj", "due_date": "" },
    { "task": "Write the ingest-lag incident retro and send to eng-wide.", "owner": "Sam", "due_date": "2026-04-27" },
    { "task": "Pair with Sam to ship the dead-letter queue alert.", "owner": "Nina", "due_date": "2026-04-24" }
  ],
  "open_questions": [
    "How should we handle backfills for the new billing schema?"
  ]
}
```

- [ ] **Step 6: Create src/lib/__tests__/fixtures/golden/customer-call.json**

```json
{
  "summary": "Sarah ran discovery with Jordan at Acme Corp, a 42-person team struggling with fragmented meeting notes. Acme had tried Otter and abandoned it. Sarah positioned MeetingScribe's structured output as the differentiator and agreed to send pricing and pilot paperwork by tomorrow. Security review and a Linear integration were flagged.",
  "decisions": [
    "Offer Acme a 30-day no-cost pilot for up to 10 users.",
    "Sarah will send pricing and pilot paperwork by the next day."
  ],
  "action_items": [
    { "task": "Send pilot paperwork, pricing, and DPA to Jordan.", "owner": "Sarah", "due_date": "2026-04-19" },
    { "task": "Loop Acme's security team into the DPA review.", "owner": "Jordan", "due_date": "" }
  ],
  "open_questions": [
    "Does a Zapier bridge cover the Linear integration need until Q3?",
    "When will native Linear integration ship?"
  ]
}
```

- [ ] **Step 7: Verify typecheck**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 8: Commit**

```bash
git add src/lib/samples src/lib/__tests__/fixtures/golden
git commit -m "feat: three sample transcripts + golden fixtures"
```

---

## Task 17: Header component + landing Hero

**Files:**
- Create: `src/components/Header.tsx`
- Create: `src/components/landing/Hero.tsx`
- Modify: `src/app/page.tsx`

- [ ] **Step 1: Create src/components/Header.tsx**

```tsx
import Link from "next/link";
import { Github } from "lucide-react";

export function Header() {
  return (
    <header className="border-b border-zinc-900">
      <div className="mx-auto flex h-14 max-w-5xl items-center justify-between px-6">
        <Link href="/" className="text-sm font-mono tracking-tight">
          <span className="text-zinc-500">meeting</span>
          <span className="text-zinc-100">scribe</span>
        </Link>
        <div className="flex items-center gap-4 text-sm">
          <Link href="/app" className="text-zinc-400 hover:text-zinc-100">
            app
          </Link>
          <a
            href="https://github.com/bright-nwokoro/meetingscribe"
            target="_blank"
            rel="noreferrer"
            className="text-zinc-400 hover:text-zinc-100"
            aria-label="GitHub"
          >
            <Github className="h-4 w-4" />
          </a>
        </div>
      </div>
    </header>
  );
}
```

- [ ] **Step 2: Create src/components/landing/Hero.tsx**

```tsx
import Link from "next/link";
import { Button } from "@/components/ui/button";
import { ArrowRight } from "lucide-react";

export function Hero() {
  return (
    <section className="mx-auto max-w-3xl px-6 pt-24 pb-16 md:pt-32">
      <h1 className="text-4xl md:text-6xl font-semibold tracking-tight text-zinc-100 leading-[1.05]">
        Parseable meeting summaries.
        <br />
        <span className="text-zinc-500">Not prose.</span>
      </h1>
      <p className="mt-6 max-w-xl text-lg text-zinc-400 leading-relaxed">
        Paste a transcript; get structured JSON with decisions, action items, and open questions —
        guaranteed to match a schema your pipeline can trust.
      </p>
      <div className="mt-10 flex items-center gap-3">
        <Button asChild size="lg">
          <Link href="/app">
            Try the demo <ArrowRight className="h-4 w-4" />
          </Link>
        </Button>
        <Button asChild variant="ghost" size="lg">
          <a href="https://github.com/bright-nwokoro/meetingscribe" target="_blank" rel="noreferrer">
            View source
          </a>
        </Button>
      </div>
    </section>
  );
}
```

- [ ] **Step 3: Wire into src/app/page.tsx**

```tsx
import { Header } from "@/components/Header";
import { Hero } from "@/components/landing/Hero";

export default function Home() {
  return (
    <>
      <Header />
      <Hero />
    </>
  );
}
```

- [ ] **Step 4: Verify visually**

Run: `pnpm dev`
Open: http://localhost:3000
Expected: dark page, large headline, two buttons, sparse header, no console errors.
Kill dev server after verifying.

- [ ] **Step 5: Commit**

```bash
git add src/components/Header.tsx src/components/landing/Hero.tsx src/app/page.tsx
git commit -m "feat(ui): sparse header and landing hero"
```

---

## Task 18: WhyStructured section + safe JsonPreview

**Files:**
- Create: `src/components/landing/JsonPreview.tsx`
- Create: `src/components/landing/WhyStructured.tsx`
- Modify: `src/app/page.tsx`

The JSON preview uses React nodes (no `dangerouslySetInnerHTML`) to tokenize and highlight the JSON. Content is entirely static but we use React nodes anyway for defense-in-depth.

- [ ] **Step 1: Create src/components/landing/JsonPreview.tsx**

```tsx
import { Fragment } from "react";

const SAMPLE = {
  summary: "Team aligned on the Q2 launch. Scope cut on presence indicators to hit June 15.",
  decisions: ["Ship collaborative editing by June 15 without presence indicators."],
  action_items: [
    { task: "Write the data-model migration plan.", owner: "Alice", due_date: "2026-04-30" },
  ],
  open_questions: ["How should we price this for enterprise?"],
};

/**
 * Tokenizes a pretty-printed JSON string into highlighted React spans.
 * Handles four classes: key, string, number, and literal (true/false/null).
 * Everything else is rendered as plain text.
 */
function tokenize(json: string) {
  const re =
    /("(?:[^"\\]|\\.)*")(\s*:)|("(?:[^"\\]|\\.)*")|\b(true|false|null)\b|(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)/g;
  const nodes: React.ReactNode[] = [];
  let lastIndex = 0;
  let key = 0;

  for (const m of json.matchAll(re)) {
    const start = m.index ?? 0;
    if (start > lastIndex) nodes.push(json.slice(lastIndex, start));

    if (m[1] && m[2]) {
      nodes.push(
        <Fragment key={key++}>
          <span className="text-accent">{m[1]}</span>
          {m[2]}
        </Fragment>,
      );
    } else if (m[3]) {
      nodes.push(
        <span key={key++} className="text-zinc-200">
          {m[3]}
        </span>,
      );
    } else if (m[4] || m[5]) {
      nodes.push(
        <span key={key++} className="text-zinc-400">
          {m[4] ?? m[5]}
        </span>,
      );
    }
    lastIndex = start + m[0].length;
  }

  if (lastIndex < json.length) nodes.push(json.slice(lastIndex));
  return nodes;
}

export function JsonPreview() {
  const pretty = JSON.stringify(SAMPLE, null, 2);
  return (
    <pre className="mt-8 overflow-x-auto rounded-md border border-zinc-800 bg-zinc-900/60 p-5 font-mono text-[13px] leading-relaxed text-zinc-300">
      <code>{tokenize(pretty)}</code>
    </pre>
  );
}
```

- [ ] **Step 2: Create src/components/landing/WhyStructured.tsx**

```tsx
import { JsonPreview } from "./JsonPreview";

export function WhyStructured() {
  return (
    <section className="mx-auto max-w-3xl px-6 py-20 border-t border-zinc-900">
      <p className="text-xs uppercase tracking-widest text-zinc-500">Why structured output</p>
      <h2 className="mt-3 text-2xl md:text-3xl font-semibold tracking-tight">
        Free-form prompts rot. Schemas don't.
      </h2>
      <p className="mt-4 max-w-2xl text-zinc-400 leading-relaxed">
        MeetingScribe uses Anthropic tool-calling and OpenAI strict JSON schemas to force the model
        through a fixed output shape. Swap models, change providers, reprompt — your downstream never
        breaks on a weird model mood.
      </p>
      <JsonPreview />
    </section>
  );
}
```

- [ ] **Step 3: Update src/app/page.tsx**

```tsx
import { Header } from "@/components/Header";
import { Hero } from "@/components/landing/Hero";
import { WhyStructured } from "@/components/landing/WhyStructured";

export default function Home() {
  return (
    <>
      <Header />
      <Hero />
      <WhyStructured />
    </>
  );
}
```

- [ ] **Step 4: Verify visually**

Run: `pnpm dev`
Expected: landing shows hero + why-structured section with a monospace JSON preview. JSON keys appear in emerald, values in light gray. No console errors.

- [ ] **Step 5: Commit**

```bash
git add src/components/landing src/app/page.tsx
git commit -m "feat(ui): WhyStructured section with inline JSON preview"
```

---

## Task 19: Transcript input, sample picker, summarize button, model picker

**Files:**
- Create: `src/components/app/TranscriptInput.tsx`
- Create: `src/components/app/SamplePicker.tsx`
- Create: `src/components/app/SummarizeButton.tsx`
- Create: `src/components/app/ModelPicker.tsx`

- [ ] **Step 1: Create src/components/app/TranscriptInput.tsx**

```tsx
"use client";
import { Textarea } from "@/components/ui/textarea";
import { MAX_TRANSCRIPT_CHARS } from "@/lib/validate";

export function TranscriptInput({
  value,
  onChange,
  disabled,
}: {
  value: string;
  onChange: (v: string) => void;
  disabled?: boolean;
}) {
  const pct = Math.min(100, (value.length / MAX_TRANSCRIPT_CHARS) * 100);
  const tooLong = value.length > MAX_TRANSCRIPT_CHARS;
  return (
    <div className="flex flex-col gap-2">
      <label className="text-xs uppercase tracking-widest text-zinc-500">Transcript</label>
      <Textarea
        value={value}
        onChange={(e) => onChange(e.target.value)}
        disabled={disabled}
        placeholder="Paste a meeting transcript, or pick a sample below…"
        className="min-h-[22rem]"
      />
      <div className="flex items-center justify-between text-xs text-zinc-500">
        <span>
          {value.length.toLocaleString()} / {MAX_TRANSCRIPT_CHARS.toLocaleString()} chars
        </span>
        <span className={tooLong ? "text-red-400" : "text-zinc-600"}>
          {tooLong ? "too long" : `${pct.toFixed(0)}%`}
        </span>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create src/components/app/SamplePicker.tsx**

```tsx
"use client";
import { samples, type SampleId } from "@/lib/samples";
import { cn } from "@/lib/utils";

export function SamplePicker({
  onPick,
  active,
}: {
  onPick: (id: SampleId) => void;
  active: SampleId | null;
}) {
  return (
    <div className="flex flex-wrap gap-2">
      {samples.map((s) => (
        <button
          key={s.id}
          type="button"
          onClick={() => onPick(s.id)}
          className={cn(
            "rounded-sm border px-3 py-1 text-xs transition-colors",
            active === s.id
              ? "border-accent/50 bg-accent/10 text-accent"
              : "border-zinc-800 text-zinc-400 hover:border-zinc-700 hover:text-zinc-200",
          )}
        >
          {s.label}
        </button>
      ))}
    </div>
  );
}
```

- [ ] **Step 3: Create src/components/app/SummarizeButton.tsx**

```tsx
"use client";
import { Button } from "@/components/ui/button";
import { ArrowRight, StopCircle } from "lucide-react";

export function SummarizeButton({
  status,
  disabled,
  onStart,
  onAbort,
}: {
  status: "idle" | "streaming" | "done" | "error";
  disabled?: boolean;
  onStart: () => void;
  onAbort: () => void;
}) {
  if (status === "streaming") {
    return (
      <Button variant="ghost" size="lg" onClick={onAbort}>
        <StopCircle className="h-4 w-4" /> Stop
      </Button>
    );
  }
  return (
    <Button size="lg" onClick={onStart} disabled={disabled}>
      {status === "done" ? "Summarize again" : "Summarize"}
      <ArrowRight className="h-4 w-4" />
    </Button>
  );
}
```

- [ ] **Step 4: Create src/components/app/ModelPicker.tsx**

```tsx
"use client";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";

export function ModelPicker({
  value,
  onChange,
}: {
  value: string;
  onChange: (v: string) => void;
}) {
  return (
    <div className="flex items-center gap-2 text-xs">
      <span className="text-zinc-500">model</span>
      <Select value={value} onValueChange={onChange}>
        <SelectTrigger className="h-7 w-[220px] text-xs">
          <SelectValue />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="claude-sonnet-4-6">claude-sonnet-4-6</SelectItem>
          <SelectItem value="gpt-4o">gpt-4o</SelectItem>
        </SelectContent>
      </Select>
    </div>
  );
}
```

- [ ] **Step 5: Verify typecheck**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 6: Commit**

```bash
git add src/components/app/TranscriptInput.tsx src/components/app/SamplePicker.tsx src/components/app/SummarizeButton.tsx src/components/app/ModelPicker.tsx
git commit -m "feat(ui): transcript input, sample picker, model picker, summarize button"
```

---

## Task 20: Key dialog (BYO-key, localStorage)

**Files:**
- Create: `src/lib/hooks/useLocalKeys.ts`
- Create: `src/components/app/KeyDialog.tsx`

- [ ] **Step 1: Create src/lib/hooks/useLocalKeys.ts**

```ts
"use client";
import { useCallback, useEffect, useState } from "react";

const STORAGE_KEY = "ms.keys.v1";

export type LocalKeys = { anthropic: string; openai: string };
const EMPTY: LocalKeys = { anthropic: "", openai: "" };

export function useLocalKeys() {
  const [keys, setKeys] = useState<LocalKeys>(EMPTY);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) setKeys({ ...EMPTY, ...JSON.parse(raw) });
    } catch {
      /* ignore */
    }
    setLoaded(true);
  }, []);

  const save = useCallback((next: LocalKeys) => {
    setKeys(next);
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(next));
    } catch {
      /* ignore */
    }
  }, []);

  const clear = useCallback(() => {
    setKeys(EMPTY);
    try {
      localStorage.removeItem(STORAGE_KEY);
    } catch {
      /* ignore */
    }
  }, []);

  const keyFor = useCallback(
    (model: string): string | undefined => {
      if (model.startsWith("claude-")) return keys.anthropic.trim() || undefined;
      if (model.startsWith("gpt-")) return keys.openai.trim() || undefined;
      return undefined;
    },
    [keys],
  );

  return {
    keys,
    loaded,
    save,
    clear,
    keyFor,
    hasAny: !!(keys.anthropic.trim() || keys.openai.trim()),
  };
}
```

- [ ] **Step 2: Create src/components/app/KeyDialog.tsx**

```tsx
"use client";
import { useState } from "react";
import { Settings } from "lucide-react";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { useLocalKeys } from "@/lib/hooks/useLocalKeys";

export function KeyDialog() {
  const { keys, save, clear } = useLocalKeys();
  const [draft, setDraft] = useState(keys);
  const [open, setOpen] = useState(false);

  return (
    <Dialog
      open={open}
      onOpenChange={(o) => {
        setOpen(o);
        if (o) setDraft(keys);
      }}
    >
      <DialogTrigger asChild>
        <Button variant="ghost" size="sm">
          <Settings className="h-3.5 w-3.5" />
          API keys
        </Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Bring your own key</DialogTitle>
          <DialogDescription>
            Keys stay in your browser (localStorage). They're sent server-side only when you
            summarize — used once to call the provider, then discarded.
          </DialogDescription>
        </DialogHeader>
        <div className="flex flex-col gap-3 pt-2">
          <label className="flex flex-col gap-1 text-sm">
            <span className="text-zinc-400">Anthropic API key</span>
            <input
              type="password"
              value={draft.anthropic}
              onChange={(e) => setDraft({ ...draft, anthropic: e.target.value })}
              placeholder="sk-ant-…"
              className="h-9 rounded-md border border-zinc-800 bg-zinc-900 px-3 font-mono text-xs focus:outline-none focus:ring-1 focus:ring-accent"
            />
          </label>
          <label className="flex flex-col gap-1 text-sm">
            <span className="text-zinc-400">OpenAI API key</span>
            <input
              type="password"
              value={draft.openai}
              onChange={(e) => setDraft({ ...draft, openai: e.target.value })}
              placeholder="sk-…"
              className="h-9 rounded-md border border-zinc-800 bg-zinc-900 px-3 font-mono text-xs focus:outline-none focus:ring-1 focus:ring-accent"
            />
          </label>
        </div>
        <div className="flex items-center justify-between pt-3">
          <Button
            variant="ghost"
            size="sm"
            onClick={() => {
              clear();
              setDraft({ anthropic: "", openai: "" });
            }}
          >
            Clear
          </Button>
          <Button
            size="sm"
            onClick={() => {
              save(draft);
              setOpen(false);
            }}
          >
            Save
          </Button>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 3: Verify typecheck and build**

Run: `pnpm typecheck && pnpm build`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
git add src/lib/hooks/useLocalKeys.ts src/components/app/KeyDialog.tsx
git commit -m "feat(ui): BYO-key dialog backed by localStorage"
```

---

## Task 21: Summary rendering — card, field, caret, list components

**Files:**
- Create: `src/components/app/StreamingCaret.tsx`
- Create: `src/components/app/SummaryField.tsx`
- Create: `src/components/app/DecisionsList.tsx`
- Create: `src/components/app/ActionItemRow.tsx`
- Create: `src/components/app/ActionItemsList.tsx`
- Create: `src/components/app/OpenQuestionsList.tsx`
- Create: `src/components/app/SummaryCard.tsx`

- [ ] **Step 1: Create src/components/app/StreamingCaret.tsx**

```tsx
export function StreamingCaret() {
  return (
    <span
      aria-hidden
      className="inline-block ml-0.5 h-[0.9em] w-[2px] translate-y-0.5 bg-accent align-middle animate-blink"
    />
  );
}
```

- [ ] **Step 2: Create src/components/app/SummaryField.tsx**

```tsx
export function SummaryField({
  label,
  streaming,
  empty,
  children,
}: {
  label: string;
  streaming: boolean;
  empty: boolean;
  children: React.ReactNode;
}) {
  return (
    <div>
      <div className="flex items-center gap-2 mb-2">
        <span className="text-xs uppercase tracking-widest text-zinc-500">{label}</span>
      </div>
      {empty && !streaming ? (
        <p className="text-sm text-zinc-600 italic">—</p>
      ) : empty && streaming ? (
        <div className="h-5 w-full animate-pulse rounded bg-zinc-900" />
      ) : (
        children
      )}
    </div>
  );
}
```

- [ ] **Step 3: Create src/components/app/DecisionsList.tsx**

```tsx
export function DecisionsList({ items }: { items: string[] }) {
  return (
    <ul className="flex flex-col gap-2">
      {items.map((d, i) => (
        <li key={i} className="flex gap-3 text-sm leading-relaxed">
          <span className="mt-2 h-1 w-1 shrink-0 rounded-full bg-accent" />
          <span className="text-zinc-200">{d}</span>
        </li>
      ))}
    </ul>
  );
}
```

- [ ] **Step 4: Create src/components/app/ActionItemRow.tsx**

```tsx
import { Badge } from "@/components/ui/badge";

export function ActionItemRow({
  task,
  owner,
  dueDate,
}: {
  task?: string;
  owner?: string;
  dueDate?: string;
}) {
  return (
    <div className="rounded-md border border-zinc-800 bg-zinc-900/40 p-3">
      <p className="text-sm text-zinc-100 leading-snug">{task ?? ""}</p>
      <div className="mt-2 flex items-center gap-2">
        {owner && (
          <Badge variant={owner === "unassigned" ? "muted" : "accent"}>@{owner}</Badge>
        )}
        {dueDate && <Badge variant="default">due {dueDate}</Badge>}
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Create src/components/app/ActionItemsList.tsx**

```tsx
import { ActionItemRow } from "./ActionItemRow";

export function ActionItemsList({
  items,
}: {
  items: Array<{ task?: string; owner?: string; due_date?: string }>;
}) {
  return (
    <div className="flex flex-col gap-2">
      {items.map((a, i) => (
        <ActionItemRow key={i} task={a.task} owner={a.owner} dueDate={a.due_date} />
      ))}
    </div>
  );
}
```

- [ ] **Step 6: Create src/components/app/OpenQuestionsList.tsx**

```tsx
export function OpenQuestionsList({ items }: { items: string[] }) {
  return (
    <ul className="flex flex-col gap-2">
      {items.map((q, i) => (
        <li key={i} className="flex gap-3 text-sm leading-relaxed">
          <span className="mt-2 h-1 w-1 shrink-0 rounded-full bg-zinc-500" />
          <span className="text-zinc-300">{q}</span>
        </li>
      ))}
    </ul>
  );
}
```

- [ ] **Step 7: Create src/components/app/SummaryCard.tsx**

```tsx
import type { PartialSummary } from "@/lib/llm/types";
import type { Summary } from "@/lib/llm/schema";
import { SummaryField } from "./SummaryField";
import { DecisionsList } from "./DecisionsList";
import { ActionItemsList } from "./ActionItemsList";
import { OpenQuestionsList } from "./OpenQuestionsList";
import { StreamingCaret } from "./StreamingCaret";

export function SummaryCard({
  partial,
  final,
  streaming,
}: {
  partial: PartialSummary;
  final: Summary | null;
  streaming: boolean;
}) {
  const s = final ?? partial;
  const decisions = s?.decisions ?? [];
  const actions = s?.action_items ?? [];
  const questions = s?.open_questions ?? [];
  const summaryText = s?.summary ?? "";

  return (
    <div className="flex flex-col gap-8 rounded-lg border border-zinc-800 bg-zinc-900/30 p-6">
      <SummaryField label="Summary" streaming={streaming} empty={!summaryText}>
        <p className="text-sm leading-relaxed text-zinc-100">
          {summaryText}
          {streaming && !final && <StreamingCaret />}
        </p>
      </SummaryField>

      <SummaryField label="Decisions" streaming={streaming} empty={decisions.length === 0}>
        <DecisionsList items={decisions} />
      </SummaryField>

      <SummaryField label="Action items" streaming={streaming} empty={actions.length === 0}>
        <ActionItemsList items={actions} />
      </SummaryField>

      <SummaryField label="Open questions" streaming={streaming} empty={questions.length === 0}>
        <OpenQuestionsList items={questions} />
      </SummaryField>
    </div>
  );
}
```

- [ ] **Step 8: Verify typecheck**

Run: `pnpm typecheck`
Expected: passes.

- [ ] **Step 9: Commit**

```bash
git add src/components/app/StreamingCaret.tsx src/components/app/SummaryField.tsx src/components/app/DecisionsList.tsx src/components/app/ActionItemRow.tsx src/components/app/ActionItemsList.tsx src/components/app/OpenQuestionsList.tsx src/components/app/SummaryCard.tsx
git commit -m "feat(ui): summary card with streaming-aware fields"
```

---

## Task 22: Export menu

**Files:**
- Create: `src/components/app/ExportMenu.tsx`

- [ ] **Step 1: Create src/components/app/ExportMenu.tsx**

```tsx
"use client";
import { useState } from "react";
import { Check, FileJson, FileText } from "lucide-react";
import { Button } from "@/components/ui/button";
import { toMarkdown } from "@/lib/export";
import type { Summary } from "@/lib/llm/schema";

export function ExportMenu({ summary }: { summary: Summary }) {
  const [copied, setCopied] = useState<"json" | "md" | null>(null);

  const copy = async (payload: string, label: "json" | "md") => {
    try {
      await navigator.clipboard.writeText(payload);
      setCopied(label);
      setTimeout(() => setCopied(null), 1500);
    } catch {
      /* ignore */
    }
  };

  return (
    <div className="flex items-center gap-2">
      <Button
        variant="ghost"
        size="sm"
        onClick={() => copy(JSON.stringify(summary, null, 2), "json")}
      >
        {copied === "json" ? (
          <Check className="h-3.5 w-3.5 text-accent" />
        ) : (
          <FileJson className="h-3.5 w-3.5" />
        )}
        JSON
      </Button>
      <Button variant="ghost" size="sm" onClick={() => copy(toMarkdown(summary), "md")}>
        {copied === "md" ? (
          <Check className="h-3.5 w-3.5 text-accent" />
        ) : (
          <FileText className="h-3.5 w-3.5" />
        )}
        Markdown
      </Button>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add src/components/app/ExportMenu.tsx
git commit -m "feat(ui): export menu (copy JSON / Markdown)"
```

---

## Task 23: Wire /app page

**Files:**
- Create: `src/app/app/page.tsx`

- [ ] **Step 1: Create src/app/app/page.tsx**

```tsx
"use client";
import { useState } from "react";
import { AlertCircle } from "lucide-react";
import { Header } from "@/components/Header";
import { TranscriptInput } from "@/components/app/TranscriptInput";
import { SamplePicker } from "@/components/app/SamplePicker";
import { SummarizeButton } from "@/components/app/SummarizeButton";
import { ModelPicker } from "@/components/app/ModelPicker";
import { KeyDialog } from "@/components/app/KeyDialog";
import { SummaryCard } from "@/components/app/SummaryCard";
import { ExportMenu } from "@/components/app/ExportMenu";
import { useSummaryStream } from "@/lib/hooks/useSummaryStream";
import { useLocalKeys } from "@/lib/hooks/useLocalKeys";
import { samples, type SampleId } from "@/lib/samples";
import { MAX_TRANSCRIPT_CHARS } from "@/lib/validate";

export default function AppPage() {
  const [transcript, setTranscript] = useState("");
  const [model, setModel] = useState("claude-sonnet-4-6");
  const [activeSample, setActiveSample] = useState<SampleId | null>(null);
  const { keyFor, hasAny } = useLocalKeys();
  const { partial, final, error, status, start, abort } = useSummaryStream();

  const pickSample = (id: SampleId) => {
    const s = samples.find((x) => x.id === id);
    if (!s) return;
    setTranscript(s.transcript);
    setActiveSample(id);
  };

  const onChangeTranscript = (v: string) => {
    setTranscript(v);
    setActiveSample(null);
  };

  const tooLong = transcript.length > MAX_TRANSCRIPT_CHARS;
  const canStart = transcript.trim().length > 0 && !tooLong && status !== "streaming";

  return (
    <>
      <Header />
      <main className="mx-auto max-w-7xl px-6 py-10">
        <div className="grid gap-8 md:grid-cols-2">
          <section className="flex flex-col gap-4">
            <TranscriptInput
              value={transcript}
              onChange={onChangeTranscript}
              disabled={status === "streaming"}
            />
            <SamplePicker onPick={pickSample} active={activeSample} />
            <div className="mt-2 flex items-center justify-between">
              <SummarizeButton
                status={status}
                disabled={!canStart}
                onStart={() => start({ transcript, model, byoKey: keyFor(model) })}
                onAbort={abort}
              />
              <div className="flex items-center gap-4">
                <ModelPicker value={model} onChange={setModel} />
                <KeyDialog />
              </div>
            </div>
            <p className="text-xs text-zinc-500">
              {hasAny
                ? "Using your key · no rate limit"
                : "Free tier · 10 summaries / day per IP. Add your key for unlimited."}
            </p>
          </section>

          <section className="flex flex-col gap-4">
            {error && (
              <div className="flex items-start gap-2 rounded-md border border-red-900 bg-red-950/30 p-3 text-sm text-red-300">
                <AlertCircle className="mt-0.5 h-4 w-4 shrink-0" />
                <div>
                  <p className="font-medium">{error.message}</p>
                  {error.code && <p className="text-xs text-red-400/70">code: {error.code}</p>}
                </div>
              </div>
            )}
            <SummaryCard partial={partial} final={final} streaming={status === "streaming"} />
            {final && (
              <div className="flex items-center justify-between text-xs text-zinc-500">
                <span>
                  {final._meta?.provider} · {final._meta?.model} ·{" "}
                  {final._meta?.usage.output ?? 0} out tokens
                </span>
                <ExportMenu summary={final} />
              </div>
            )}
          </section>
        </div>
      </main>
    </>
  );
}
```

- [ ] **Step 2: Verify build**

Run: `pnpm build`
Expected: builds cleanly, no type errors.

- [ ] **Step 3: Local smoke test with a real key (optional)**

If an Anthropic key is available locally:

```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." >> .env.local
pnpm dev
```

Open http://localhost:3000/app, click the "Product kickoff" sample chip, click Summarize. Confirm fields populate progressively. Kill dev server.

If no key, skip — verify later in Vercel preview.

- [ ] **Step 4: Commit**

```bash
git add src/app/app/page.tsx
git commit -m "feat(app): wire /app page with streaming hook and all controls"
```

---

## Task 24: README rewrite + LICENSE + .env.example

**Files:**
- Modify: `README.md`
- Create: `LICENSE`
- Create: `.env.example`

- [ ] **Step 1: Create LICENSE**

```
MIT License

Copyright (c) 2026 Bright Nwokoro

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 2: Create .env.example**

```
# LLM providers (at least one required server-side — users can also BYO from the UI)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# Optional model overrides
ANTHROPIC_MODEL=claude-sonnet-4-6
OPENAI_MODEL=gpt-4o-2024-08-06

# Optional rate limiting (if unset, rate limiting is disabled — owner bears any abuse)
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
```

- [ ] **Step 3: Overwrite README.md**

Replace the entire contents of `README.md` with this:

````markdown
# MeetingScribe

> Parseable meeting summaries. Not prose.

Paste a transcript; get structured JSON with decisions, action items, and open questions — guaranteed to match a schema your pipeline can trust.

**🔗 Live demo:** https://meetingscribe.brightnwokoro.dev
**👤 Built by:** [Bright Nwokoro](https://brightnwokoro.dev) · [hello@brightnwokoro.dev](mailto:hello@brightnwokoro.dev)

![MeetingScribe demo](./docs/demo.gif)

---

## What this repo is (and isn't)

A focused proof-of-craft around **structured LLM output**: one JSON Schema drives two providers (Anthropic Claude, OpenAI), and partial JSON streams to the UI in real time. It is intentionally stateless — no database, no auth — so the architecture stays legible and the live demo costs nothing to run.

Roadmap items (audio transcription, persistence, email delivery, self-host stack) are real and will be added in follow-ups; they are deliberately deferred so the core artifact can ship polished.

## Why this exists

Every team wants AI meeting notes. The hosted leaders (Otter, Fireflies, Fathom) charge $20+/user/month and hold transcripts on their infrastructure. Many teams — legal, healthcare, founder-mode startups with sensitive product discussions — can't or won't send meetings to a SaaS.

MeetingScribe is the open, self-hostable alternative. The structured output is the point: downstream workflows (Linear, Jira, Slack, email) can consume the summary directly without fragile string parsing.

The senior pattern it demonstrates: *forcing* the LLM through a JSON schema (Anthropic tool-calling, OpenAI strict structured outputs) so the output is always valid and your pipeline never breaks on a weird model mood.

## What it does

- **Input:** paste a meeting transcript (up to 50K characters)
- **Extraction:** Claude Sonnet 4.6 (default) or GPT-4o, via a shared JSON Schema
- **Streaming:** partial JSON streams into the UI field-by-field as the model generates
- **Output:** structured summary with
  - `summary` — 3–5 sentence executive overview
  - `decisions[]` — things the meeting settled
  - `action_items[]` — `{ task, owner, due_date? }` tuples
  - `open_questions[]` — unresolved items
- **Export:** copy as JSON or Markdown
- **Hybrid demo:** 3 canned sample transcripts work out of the box; users paste their own API key (stored in `localStorage`) to summarize custom transcripts without the rate limit

## Architecture

```
  ┌──────────┐  POST /api/summarize   ┌────────────────┐
  │  Next.js │ ────(transcript)─────▶ │ /api/summarize │
  │   /app   │ ◀──(SSE partials)───── │  Edge runtime  │
  └──────────┘                        └────────┬───────┘
                                               │
                                   ┌───────────┴────────────┐
                                   │       Adapter          │
                                   │   (one interface,      │
                                   │    one JSON schema)    │
                                   └────┬──────────────┬────┘
                                        │              │
                                  ┌─────▼─────┐  ┌─────▼─────┐
                                  │  Claude   │  │  OpenAI   │
                                  │ tool_use  │  │ json_schema│
                                  └───────────┘  └───────────┘
```

## Stack

| Layer           | Tech                                                                 |
| --------------- | -------------------------------------------------------------------- |
| Frontend        | Next.js 15 (App Router), React 19, TypeScript, Tailwind, shadcn/ui   |
| API             | Next.js Edge runtime + Server-Sent Events                            |
| Extraction      | Claude Sonnet 4.6 (Anthropic) · GPT-4o (OpenAI) · same JSON Schema   |
| Streaming parse | `partial-json` incremental JSON parsing                              |
| Rate limiting   | Upstash Redis (optional; no-op fallback)                             |
| Tests           | Vitest with mocked provider streams + golden fixtures                |
| Deploy          | Vercel                                                               |

## Quick start

```bash
git clone https://github.com/bright-nwokoro/meetingscribe
cd meetingscribe

cp .env.example .env.local
# fill at least ANTHROPIC_API_KEY or OPENAI_API_KEY

pnpm install
pnpm dev
# → http://localhost:3000
```

## Environment variables

```bash
# At least one of these is required on the server — users can also BYO
# from the UI's Settings dialog.
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...

# Optional model overrides
ANTHROPIC_MODEL=claude-sonnet-4-6
OPENAI_MODEL=gpt-4o-2024-08-06

# Optional rate limiting (if unset, limiting is disabled).
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
```

## How structured extraction works

Both providers are invoked through adapters that implement a single interface:

```ts
interface LLMAdapter {
  name: "claude" | "openai";
  stream(args: { transcript: string; apiKey: string; signal?: AbortSignal }):
    AsyncGenerator<{ partial: PartialSummary }, StreamResult, void>;
}
```

### Claude side (tool-calling)

```ts
await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  body: JSON.stringify({
    model: "claude-sonnet-4-6",
    stream: true,
    tools: [{ name: "record_summary", input_schema: summaryJsonSchema }],
    tool_choice: { type: "tool", name: "record_summary" },
    messages: [{ role: "user", content: transcript }],
  }),
});
// Stream emits input_json_delta events; we append and parse with partial-json.
```

### OpenAI side (strict structured outputs)

```ts
await fetch("https://api.openai.com/v1/chat/completions", {
  method: "POST",
  body: JSON.stringify({
    model: "gpt-4o-2024-08-06",
    stream: true,
    stream_options: { include_usage: true },
    response_format: {
      type: "json_schema",
      json_schema: { name: "record_summary", strict: true, schema: summaryJsonSchema },
    },
    messages: [...],
  }),
});
// Stream emits delta.content fragments; same partial-json parse.
```

**The schema is defined once** in `src/lib/llm/schema.ts` and consumed by both. Changing the shape forces both adapters' golden tests to fail — a deliberate constraint.

## Project structure

```
src/
├── app/
│   ├── page.tsx                 # landing
│   ├── app/page.tsx             # tool
│   └── api/summarize/route.ts   # Edge SSE endpoint
├── components/
│   ├── landing/                 # Hero, WhyStructured, JsonPreview
│   ├── app/                     # TranscriptInput, SummaryCard, KeyDialog, …
│   └── ui/                      # shadcn primitives
└── lib/
    ├── llm/
    │   ├── schema.ts            # JSON Schema + Zod (source of truth)
    │   ├── anthropic.ts         # Claude adapter
    │   ├── openai.ts            # OpenAI adapter
    │   └── prompt.ts            # shared system prompt
    ├── hooks/useSummaryStream.ts
    ├── samples/                 # 3 canned transcripts
    └── __tests__/               # Vitest + golden fixtures
```

## Testing

```bash
pnpm test           # schema, partial-json, both adapters, validator, export
pnpm typecheck
pnpm lint
```

Adapter tests use **mocked provider streams** against **golden fixtures**. If you change the summary shape, the goldens fail loudly — update them deliberately.

## Roadmap

- [ ] Audio upload + Whisper transcription (cloud + local `whisper.cpp`)
- [ ] Supabase-backed persistence (auth, meeting history)
- [ ] Email delivery via Resend
- [ ] Speaker diarization (pyannote + Whisper)
- [ ] Retrieval over past meetings (pgvector)
- [ ] Self-host Docker Compose stack
- [ ] Slack bot (`/scribe`)

These are deliberately deferred — see [What this repo is](#what-this-repo-is-and-isnt).

## Contributing

PRs welcome. Adapter tests in `src/lib/__tests__/` use golden fixtures — if you change the summary shape, update `src/lib/__tests__/fixtures/golden/*.json`.

## License

MIT — see [LICENSE](LICENSE).

## Contact

Freelance AI engineering — RAG, chat widgets, AI copilots, end-to-end.
**Email:** hello@brightnwokoro.dev
**Portfolio:** https://brightnwokoro.dev
**Book a call:** https://calendly.com/brightnwokoro/intro
````

- [ ] **Step 4: Commit**

```bash
git add README.md LICENSE .env.example
git commit -m "docs: rewrite README to match shipped feature set, add LICENSE and .env.example"
```

---

## Task 25: GitHub Actions CI workflow

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Create .github/workflows/ci.yml**

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9.12.0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
        env:
          SKIP_ENV_VALIDATION: "true"
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "chore: add GitHub Actions CI (lint, typecheck, test, build)"
```

---

## Task 26: Final verification and push

No new files.

- [ ] **Step 1: Full lint + typecheck + test + build**

Run: `pnpm lint && pnpm typecheck && pnpm test && pnpm build`
Expected: all green.

- [ ] **Step 2: Manual smoke test against a real provider (optional)**

If Anthropic or OpenAI key available:

```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." >> .env.local
pnpm dev
```

Open http://localhost:3000/app, pick "Product kickoff" sample, click Summarize. Confirm:
- Summary populates progressively (not all-at-once after a pause).
- Blinking emerald caret appears during streaming.
- Decisions arrive one at a time.
- Action items arrive as complete rows.
- Final card shows provider/model/token counts.
- Copy JSON → paste into a JSON validator → valid.
- Copy Markdown → paste into a markdown renderer → readable.
- Abort mid-stream stops generation, caret disappears.

If no key, skip.

- [ ] **Step 3: Responsive check**

In dev, resize browser to ~375px wide and confirm:
- Landing hero stacks cleanly.
- Tool page stacks vertically.
- No horizontal scroll.
- Buttons remain tappable.

- [ ] **Step 4: Format pass**

Run: `pnpm lint:fix`
Expected: no changes. If there are changes, review and:

```bash
git add .
git commit -m "chore: final formatting pass"
```

- [ ] **Step 5: Push**

```bash
git push origin main
```

Confirm:
- CI turns green in GitHub.
- If Vercel is connected, a preview deploy succeeds and `/app` loads.

---

## Self-review notes (for the plan author)

**Spec coverage:**

| Spec section                                        | Task(s)                       |
|-----------------------------------------------------|-------------------------------|
| 2. Scope (MVP items)                                | 1–26                          |
| 3. Architecture (routes, Edge runtime, req/resp)    | 1, 13                         |
| 4. Data flow                                        | 13                            |
| 5. LLM adapter layer                                | 5, 6, 8, 9, 10                |
| 6. Rate limiting                                    | 12                            |
| 7. UI structure + streaming UX                      | 3, 4, 17–21, 23               |
| 8. Sample transcripts                               | 16                            |
| 9. Testing strategy + goldens                       | 5, 7, 8, 9, 10, 11, 14, 16    |
| 10. Developer experience (Biome, pnpm, scripts)     | 1, 2                          |
| 11. CI / deploy                                     | 25                            |
| 12. Repo structure                                  | matches plan                  |
| 13. README revisions                                | 24                            |
| 14. Non-goals                                       | honored — none implemented    |
| 15. Open implementation questions                   | resolved: `partial-json`, Biome defaults, inline error banner |

**No placeholders.** Types and identifiers used in later tasks match their earlier definitions: `LLMAdapter`, `StreamChunk`, `StreamResult`, `PartialSummary`, `AdapterArgs`, `AdapterError`, `pickAdapter`, `resolveProviderKey`, `SummarizeRequestSchema`, `MAX_TRANSCRIPT_CHARS`, `toMarkdown`, `useSummaryStream`, `useLocalKeys`, `samples`, `SampleId`, `SummarySchema`, `ActionItemSchema`, `summaryJsonSchema`, `SYSTEM_PROMPT`, `Summary`, `ActionItem`.
