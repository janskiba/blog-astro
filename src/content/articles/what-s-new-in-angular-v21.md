---
title: "What's New in Angular v21"
summary: "Angular v21 ships zoneless change detection by default, experimental Signal Forms, Vitest as the new test runner, and a new @angular/aria package for accessibility."
publishedAt: 2026-03-05
---

## Zoneless change detection by default

In version 21, when generating a new project with `ng new`, the application is created in _zoneless mode_, without **Zone.js**. Change detection is triggered only by explicit reactive sources such as **signals** and **observables** (via the `async` pipe). This improves performance by reducing unnecessary checks and decreases the bundle size by excluding Zone.js.

This shift aligns with the continued evolution of the **OnPush mindset** and **immutable state management**.

It's important to note that you can still use Zone.js by adding the proper provider in `app.config.ts`:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [provideZoneChangeDetection({ eventCoalescing: true })],
};
```

## Signal forms

In version 21, Angular introduces **Signal Forms** in an _experimental_ state. Signal Forms aim to replace reactive forms by simplifying code and reducing boilerplate.

Here's an example showing the basic functionality:

```typescript
import { ChangeDetectionStrategy, Component, signal } from "@angular/core";
import { form, FormField, required } from "@angular/forms/signals";

interface UserData {
  name: string;
}

@Component({
  selector: "app-simple-form",
  imports: [FormField],
  template: `
    <form>
      <div>
        <label
          >Name:
          <input [formField]="form.name" />
        </label>
        @if (form.name().touched() && form.name().invalid()) {
          <span style="color: red;">
            {{ form.name().errors()[0].message }}
          </span>
        }
      </div>
      <p>Hello {{ form.name().value() }}</p>
    </form>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SimpleFormComponent {
  model = signal<UserData>({ name: "" });
  form = form(this.model, (path) => {
    required(path.name, { message: "Name is required" });
  });
}
```

**How it works:**

1. The `model` signal holds the form data as the single source of truth.
2. The `form()` function creates reactive fields with validation.
3. The `[formField]` binding automatically links form controls in the template.
4. Validation errors display automatically based on user actions.

Traditional forms are still supported. Signal Forms are part of a long-term plan and currently in a feedback phase before becoming part of the stable API.

## Vitest as test runner

Angular replaces the previous test runner, **Karma**, with **Vitest**, offering much faster startup, native TypeScript support, snapshot testing, fake timers, and parallel execution — all of which significantly improve testing feedback cycles.

There's also an experimental migration schematic for older projects:

```bash
ng g @schematics/angular:refactor-jasmine-vitest
```

## Accessibility with @angular/aria

To make it easier to build accessible UI libraries, version 21 introduces a new package — **@angular/aria**. It includes ready-made ARIA directives and patterns for complex widgets like grids, trees, menus, and autocompletes. These building blocks integrate accessibility best practices, saving developers from re-implementing every keyboard behavior and attribute manually.

## Extra small features

1. Updated CLDR library for better currency and date formatting.
2. Regular expressions in templates.
3. `SimpleChanges` now supports generics:
   ```typescript
   ngOnChanges(changes: SimpleChanges<MyComponent>) {
     const nameChange = changes['name']; // TypeScript knows its a string!
     if (nameChange.firstChange) { /* ... */ }
   }
   ```
4. `KeyValue` pipe supports optional keys:
   ```html
   @for (item of {a: 1, b: undefined} | keyvalue; track item.key) {
   {{item.key}}: {{item.value ?? 'no value'}} }
   ```
5. Utility classes for Material Design tokens.
6. And the most important — new mascot named Angie!
