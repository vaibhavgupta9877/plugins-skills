# Tailwind v4 tokens and cva patterns

Templates, not prescriptions. The **values** come from `frontend-design`'s design plan for this
specific brief; this file is only the plumbing that holds them.

---

## `@theme` — the primitive and semantic layers

`src/app/globals.css`:

```css
@import "tailwindcss";

@theme {
  /* ---- primitives: raw palette, no meaning ---- */
  --color-brand-50:  oklch(0.97 0.02 250);
  --color-brand-500: oklch(0.62 0.19 250);
  --color-brand-700: oklch(0.48 0.17 250);
  --color-neutral-0:   oklch(1    0 0);
  --color-neutral-100: oklch(0.97 0 0);
  --color-neutral-500: oklch(0.55 0 0);
  --color-neutral-900: oklch(0.20 0 0);
  --color-red-500:   oklch(0.58 0.21 25);

  /* ---- semantic: what components consume ---- */
  --color-surface:         var(--color-neutral-0);
  --color-surface-raised:  var(--color-neutral-100);
  --color-border:          oklch(0.90 0 0);
  --color-text:            var(--color-neutral-900);
  --color-text-muted:      var(--color-neutral-500);
  --color-accent:          var(--color-brand-500);
  --color-accent-hover:    var(--color-brand-700);
  --color-on-accent:       var(--color-neutral-0);
  --color-danger:          var(--color-red-500);

  /* ---- type roles ---- */
  --font-display: "…", ui-serif, Georgia, serif;
  --font-body:    "…", ui-sans-serif, system-ui, sans-serif;
  --font-mono:    "…", ui-monospace, SFMono-Regular, monospace;

  --text-display:   3.5rem;  --text-display--line-height:   1.05;
  --text-heading-1: 2.25rem; --text-heading-1--line-height: 1.15;
  --text-heading-2: 1.5rem;  --text-heading-2--line-height: 1.25;
  --text-body:      1rem;    --text-body--line-height:      1.55;
  --text-caption:   0.8125rem; --text-caption--line-height: 1.4;

  /* ---- radii, elevation, motion ---- */
  --radius-card:  0.75rem;
  --radius-input: 0.5rem;
  --shadow-raised: 0 1px 2px oklch(0 0 0 / 0.06), 0 4px 12px oklch(0 0 0 / 0.06);

  --ease-out:    cubic-bezier(0.23, 1, 0.32, 1);
  --ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);
  --duration-fast: 150ms;
  --duration-base: 250ms;
}
```

Use `oklch()` rather than hex: lightness is perceptually uniform, so generating a hover shade or a
dark-mode variant is a predictable adjustment instead of a guess.

---

## Dark mode — redefine semantics only

Never redefine primitives. Only the semantic layer changes, so every component follows for free.

```css
@custom-variant dark (&:where(.dark, .dark *));

.dark {
  --color-surface:        oklch(0.17 0.01 250);
  --color-surface-raised: oklch(0.22 0.01 250);   /* lighter, not a bigger shadow */
  --color-border:         oklch(0.30 0.01 250);
  --color-text:           oklch(0.96 0 0);
  --color-text-muted:     oklch(0.68 0 0);
  --color-accent:         oklch(0.70 0.16 250);   /* lighter + less saturated in dark */
  --color-on-accent:      oklch(0.17 0.01 250);
  --shadow-raised: 0 1px 2px oklch(0 0 0 / 0.4), 0 4px 12px oklch(0 0 0 / 0.3);
}
```

Wire the class with `next-themes` and set `suppressHydrationWarning` on `<html>`. Include
`color-scheme: light dark` so native controls and scrollbars follow.

Check contrast in **both** themes: body text 4.5:1, large text and UI boundaries 3:1.

---

## cva component pattern

```tsx
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const button = cva(
  // base: every variant shares these — including all five states
  "inline-flex items-center justify-center gap-2 rounded-[--radius-input] " +
    "font-medium select-none whitespace-nowrap " +
    "transition-[transform,background-color,opacity] duration-[--duration-fast] ease-[--ease-out] " +
    "active:scale-[0.97] " +
    "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[--color-accent] focus-visible:ring-offset-2 " +
    "disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        primary:   "bg-[--color-accent] text-[--color-on-accent] hover:bg-[--color-accent-hover]",
        secondary: "bg-[--color-surface-raised] text-[--color-text] hover:brightness-95",
        ghost:     "bg-transparent text-[--color-text] hover:bg-[--color-surface-raised]",
        danger:    "bg-[--color-danger] text-white hover:brightness-95",
      },
      size: {
        sm: "h-8  px-3 text-[--text-caption]",
        md: "h-10 px-4 text-[--text-body]",
        lg: "h-12 px-6 text-[--text-body]",
      },
    },
    defaultVariants: { variant: "primary", size: "md" },
  },
);

type ButtonProps = React.ComponentPropsWithoutRef<"button"> & VariantProps<typeof button>;

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, type = "button", ...props }, ref) => (
    <button ref={ref} type={type} className={cn(button({ variant, size }), className)} {...props} />
  ),
);
Button.displayName = "Button";
```

The four things that make this reusable rather than a fork magnet: `forwardRef`, `className` merged
last, native props spread, and a `type` default so it never accidentally submits a form.

---

## Compound component

```tsx
function Card({ className, ...p }: React.ComponentPropsWithoutRef<"div">) {
  return <div className={cn("rounded-[--radius-card] border border-[--color-border] bg-[--color-surface] shadow-[--shadow-raised]", className)} {...p} />;
}
Card.Header = function CardHeader(p: React.ComponentPropsWithoutRef<"div">) {
  return <div className={cn("p-5 pb-3", p.className)} {...p} />;
};
Card.Body = function CardBody(p: React.ComponentPropsWithoutRef<"div">) {
  return <div className={cn("p-5 pt-0", p.className)} {...p} />;
};
```

Prefer this over `<Card title="…" footer={…} headerAction={…} />`. Composition scales; configuration
props accumulate until the component has fifteen of them and no clear shape.

---

## Enforcement

Make the rules mechanical rather than remembered:

```jsonc
// .eslintrc — no raw hex in JSX/TS
"no-restricted-syntax": [
  "warn",
  { "selector": "Literal[value=/#[0-9a-fA-F]{3,8}/]", "message": "Use a semantic token." }
]
```

- `prettier-plugin-tailwindcss` for class order, so diffs stay readable.
- `eslint-plugin-tailwindcss` to catch contradictory and arbitrary classes.
- A CI grep for `style={{` and `text-[…px]` outside the token files.
- One `tokens.css` / `globals.css` as the only file allowed to contain raw values, and say so in
  `CLAUDE.md`.
