# Design System

> The design system is the shared visual language between design and development.
> It prevents inconsistencies, accelerates design, and reduces rework.
> **Rule:** Before creating a new component, check here if it already exists.

> **Source of the tokens below:** extracted visually from `mockup-auth-onboarding.png`
> (auth/onboarding screens). These are close approximations, not pixel-sampled values —
> **confirm the exact hex codes against the original Figma file** before using them in
> production code, and update this file once confirmed.

---

## Design tokens

Tokens are the design system's variables. Changing a token changes the entire system.

### Colors

```css
/* Base palette — dark theme, gold/yellow accent */
--color-bg-page:    #0D0D0D;   /* Near-black app background, seen on every screen */
--color-bg-card:    #1A1A1A;   /* Slightly lighter panels/cards */
--color-bg-input:   #1E1E1E;   /* Text input fill */
--color-border:     #3A3A3A;   /* Input/card borders on dark background */

--color-primary-500: #F2C230;  /* Gold/yellow — primary buttons ("Iniciar sesión",
                                   "Registrarme", "Continuar"), logo badge accent */
--color-primary-700: #C99A1E;  /* Darker gold — pressed/hover state (estimated) */

--color-secondary-500: #000000; /* Secondary button fill ("Crear cuenta"), black with
                                    white text/border */

/* Semantic colors — not yet visible in the mockup, proposed to match the palette
   until the team confirms them */
--color-success:  #388E3C;
--color-warning:  #F2C230;      /* reuse primary gold for warnings, avoid a second yellow */
--color-error:    #D32F2F;
--color-info:     #3B82F6;

/* Text */
--color-text-primary:   #FFFFFF; /* Titles, primary labels on dark background */
--color-text-secondary: #9CA3AF; /* Helper text, placeholders */
--color-text-on-primary: #0D0D0D; /* Text/icon color on top of the gold buttons */
--color-text-disabled:  #5A5A5A;

/* Backgrounds */
--color-bg-overlay: rgba(0,0,0,0.6);
```

### Typography

```css
/* Families — the mockup uses a clean system sans-serif; exact family not yet confirmed */
--font-family-sans:  'Inter, system-ui, sans-serif'; /* placeholder until confirmed */
--font-family-mono:  '[Mono font name], monospace';

/* Sizes (modular scale 1.25) */
--font-size-xs:   0.75rem;   /* 12px */
--font-size-sm:   0.875rem;  /* 14px */
--font-size-base: 1rem;      /* 16px */
--font-size-lg:   1.25rem;   /* 20px */
--font-size-xl:   1.563rem;  /* 25px */
--font-size-2xl:  1.953rem;  /* 31px */
--font-size-3xl:  2.441rem;  /* 39px */

/* Weights */
--font-weight-regular: 400;
--font-weight-medium:  500;
--font-weight-bold:    700;

/* Line height */
--line-height-tight:  1.2;
--line-height-normal: 1.5;
--line-height-loose:  1.8;
```

### Spacing

```css
/* 4px system */
--space-1:  0.25rem;   /* 4px */
--space-2:  0.5rem;    /* 8px */
--space-3:  0.75rem;   /* 12px */
--space-4:  1rem;      /* 16px */
--space-6:  1.5rem;    /* 24px */
--space-8:  2rem;      /* 32px */
--space-12: 3rem;      /* 48px */
--space-16: 4rem;      /* 64px */
```

### Borders and shadows

```css
/* Border radius — the mockup's buttons and inputs read as moderately rounded */
--radius-sm: 4px;
--radius-md: 8px;
--radius-lg: 16px;
--radius-full: 9999px;  /* Pill */

/* Shadows — mostly invisible on a near-black background, kept for light surfaces */
--shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
--shadow-md: 0 4px 6px rgba(0,0,0,0.1);
--shadow-lg: 0 10px 15px rgba(0,0,0,0.15);
```

---

## Components

### Buttons

Observed in the mockup: a filled gold `Primary` button used for the main call to action on
every auth screen (`Iniciar sesión`, `Registrarme`, `Continuar`), and a filled black
`Secondary` button with white text for the alternate action (`Crear cuenta`).

| Variant | Use | Disabled state |
|---------|-----|----------------|
| Primary (gold fill, dark text) | Main action on the page | `opacity: 0.5; cursor: not-allowed` |
| Secondary (black fill, white text) | Secondary actions | same |
| Danger | Destructive actions (delete) | same |
| Ghost | Tertiary actions, links | same |

**Usage rules:**
- Only one Primary action per view
- Danger only with modal confirmation ("Are you sure?")
- Buttons have a loading state for async operations

### Forms

Observed in the mockup: dark-filled inputs (`--color-bg-input`) with a subtle border, a
password field with a visibility toggle, and a step indicator (1-2-3 dots) for the
multi-step owner registration wizard.

| Component | When to use |
|-----------|-------------|
| Input text | Single-line free text |
| Textarea | Multi-line free text |
| Select | Fixed list of options (< 15 items) |
| Combobox | List with search (> 15 items or dynamic loading) |
| Checkbox | Independent binary option |
| Radio | Select one option from a few (2-5) |
| Toggle | Enable/disable a feature |
| DatePicker | Date selection |
| Step indicator | Multi-step forms (e.g. owner registration wizard) |

**Error messages in forms:**
- The message appears below the field, in red
- The field border turns red
- The message says how to fix the error, not just that there is an error

```
✓ "The email must have the format user@domain.com"
✗ "Invalid email"
```

### Feedback

| Component | When | Duration |
|-----------|------|---------|
| Toast/Snackbar | Action confirmations | 4 seconds |
| Inline alert | Form errors | Until corrected |
| Modal | Destructive confirmations, irreversible actions | Until the user decides |
| Loading spinner | Operations > 200ms | Until finished |
| Skeleton | Loading list content / cards | Until loaded |

### Data table

| Aspect | Behavior |
|--------|---------|
| Pagination | Maximum 20 rows per page (user-configurable) |
| Sorting | Click on column, toggle asc/desc |
| Filters | Side panel or filter row above the table |
| Selection | Checkbox in the first column |
| Actions | Final column with actions menu (edit, delete, etc.) |
| Empty state | Illustration + message + primary action CTA |

---

## UX patterns

### Principles

1. **Confirm before destroying:** Any action that permanently deletes or modifies data requires a confirmation modal.

2. **Immediate feedback:** Every action must have a visual response in < 100ms (even if it is just the loading state).

3. **Prevent rather than correct:** Validate in real time in the form, not only on submit.

4. **Empty state as a feature:** The screen without data is the new user's first impression — guide them to the first action.

5. **Role clarity at sign-up:** The role picker (`/register-choice`) makes the user explicitly choose client / barbershop owner / barber before anything else, since the whole app shell depends on that role.

### Error handling

| Scenario | What to show |
|----------|-------------|
| Network error | Toast "No connection. Retrying..." with automatic retry |
| 401 error | Redirect to login with message "Your session expired" |
| 403 error | Screen "You do not have permission to view this" with link to support |
| 404 error | 404 screen with back navigation |
| 500 error | Error toast + "Retry" button |
| Timeout | Toast "This is taking longer than normal" with cancel option |

---

## Accessibility guide (minimums)

| Aspect | Minimum required |
|--------|-----------------|
| Text contrast | WCAG AA (4.5:1 for normal text, 3:1 for large text) — **check gold-on-dark and white-on-dark combinations against this once the exact hex values are confirmed** |
| Keyboard navigation | All interactive elements accessible with Tab |
| Form labels | All fields with associated label (`for` / `aria-label`) |
| Images | Descriptive alt text on all non-decorative images |
| Visible focus | Visible focus indicator on all interactive elements |

---

## Correlations

- Navigation map → `12-ux-ui/navigation-map.md`
- Mockup (auth/onboarding flow) → `12-ux-ui/mockup-auth-onboarding.png`
- UX non-functional requirements → `04-requirements/non-functional.md`
