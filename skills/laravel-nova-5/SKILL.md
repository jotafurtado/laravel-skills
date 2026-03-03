---
name: laravel-nova-5
description: "Develops Laravel Nova 5 administration panels. Activates when creating or editing Nova resources, fields, actions, filters, lenses, metrics, dashboards, authorization policies, relationships, panels, tabs, repeater fields, dependent fields, custom tools, or any Nova admin panel feature. Also triggers when the user mentions 'Nova', 'admin panel', 'back-office', 'resource CRUD', 'Nova action', 'Nova metric', 'Nova lens', 'Nova filter', 'Nova dashboard', or any reference to building Laravel-based administration interfaces. Use this skill even for questions about Nova authorization, field visibility, validation in Nova context, or Nova customization like custom fields, tools, and cards."
license: MIT
metadata:
  author: community
  version: "1.0.0"
  domain: backend
  nova_version: "5.x"
  laravel_version: ">=10.x"
  php_version: ">=8.1"
  triggers: Nova, Laravel Nova, admin panel, back-office, resource CRUD, Nova action, Nova metric, Nova lens, Nova filter, Nova dashboard, Nova resource, Nova field, Nova tool, Nova card
  role: specialist
  scope: implementation
  output-format: code
---

# Laravel Nova 5 Development

## When to Apply

Activate this skill when:

- Creating or editing Nova resources, fields, or relationships
- Implementing actions, filters, lenses, or metrics
- Setting up authorization policies for Nova
- Configuring dashboards, menus, tabs, or panels
- Creating custom tools, cards, or resource tools
- Working with dependent fields, repeater fields, or file uploads

## Requirements

PHP 8.1+, Laravel 10+, Nova 5.x. Frontend: Vue 3.5, Inertia.js 2.x, Heroicons 2.x.

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Actions, Filters & Lenses | `references/actions-and-filters.md` | Actions (sync/queued/batchable), filters, lenses |
| Metrics & Dashboards | `references/metrics-and-dashboards.md` | Value, Trend, Partition, Progress, Table metrics; dashboards |
| Customization | `references/customization.md` | Custom tools, resource tools, menus, notifications, stubs |

## Constraints

### MUST DO
- Use PHP 8.1+ features (readonly, enums, typed properties)
- Type hint all method parameters and return types
- Follow Nova conventions for resource naming (singular class, plural display)
- Use `HasMany::make('Posts')` with plural names for HasMany relationships
- Add relationships to `$with` when used in `title()` or `subtitle()`
- Use `asBigInt()` on `ID` for big integer primary keys
- Implement `ShouldQueue` for heavy actions to avoid timeouts
- Run `php artisan storage:link` when using `public` disk with local driver
- Use `@import "tailwindcss"` in custom components (Nova 5 uses Tailwind v4)

### MUST NOT DO
- Use singular names for `HasMany` relationships
- Define `dependsOn` on unsupported field types (`File`, `Image`, `Audio`, `VaporFile`, `VaporImage`)
- Use `creationRules`/`updateRules` inside Repeatable fields
- Use `@tailwind` directives in custom components (deprecated in Tailwind v4)
- Forget to register policies — Nova silently allows all when no policy exists
- Skip eager loading for relationships used in resource display

---

## Resource Basics

Resources live in `app/Nova` and map to Eloquent models:

```bash
php artisan nova:resource Post
```

```php
namespace App\Nova;

use Laravel\Nova\Fields\ID;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Http\Requests\NovaRequest;

class Post extends Resource
{
    public static string $model = \App\Models\Post::class;

    public static $title = 'title';

    public static $search = ['id', 'title'];

    public function fields(NovaRequest $request): array
    {
        return [
            ID::make()->sortable(),
            Text::make('Title')->sortable()->rules('required', 'max:255'),
        ];
    }
}
```

Key resource properties:

```php
public static $with = ['user'];           // eager-load for fields/title/subtitle
public static $perPageOptions = [25, 50, 100];
public static $debounce = 0.5;            // search debounce in seconds
public static $polling = true;            // enable index polling
public static $pollingInterval = 5;
public static $showPollingToggle = true;
```

---

## Fields

Defined in `fields()` via static `make()`. Nova auto-converts names to snake_case columns:

```php
Text::make('Full Name');              // → full_name column
Text::make('Email', 'email_address'); // explicit column mapping
```

### Available Field Types

`ID`, `Text`, `Textarea`, `Boolean`, `Number`, `Select`, `Date`, `DateTime`, `Currency`, `Badge`, `Markdown`, `Trix`, `Image`, `File`, `Avatar`, `KeyValue`, `Sparkline`, `Tag`, `Hidden`.

### Visibility & Configuration

```php
Text::make('Name')
    ->sortable()->readonly()->required()->nullable()
    ->placeholder('Enter name')->help('Display name.')
    ->default('Default')->copyable()
    ->showOnIndex()->hideWhenUpdating()
    ->onlyOnForms()->exceptOnForms()
    ->canSee(fn ($request) => $request->user()->is_admin);
```

### Select

```php
Select::make('Size')->options(['S' => 'Small', 'M' => 'Medium', 'L' => 'Large'])
    ->displayUsingLabels()->searchable();
```

### Badge

```php
Badge::make('Status', fn () => match ($this->status) {
    'active' => 'Active', default => 'Inactive',
})->map(['Active' => 'success', 'Inactive' => 'danger']);
```

### Big Integer IDs

```php
ID::make()->asBigInt()->sortable();
```

---

## Computed Fields (Nova 5)

Nova 5 gives computed fields a unique `$attribute`, enabling them as dependent fields:

```php
Boolean::make('Leave a comment?', 'comment_boolean')->computed(),

Text::make('Comment')->hidden()
    ->dependsOn('comment_boolean', function (Text $field, NovaRequest $request, FormData $formData) {
        if ($formData->boolean('comment_boolean')) {
            $field->show();
        }
    }),
```

## Dependent Fields

Reactively modify field config based on other field values:

```php
Select::make('Purchase Type', 'type')
    ->options(['personal' => 'Personal', 'gift' => 'Gift']),

Text::make('Recipient')->readonly()
    ->dependsOn(['type'], function (Text $field, NovaRequest $request, FormData $formData) {
        if ($formData->type === 'gift') {
            $field->readonly(false)->rules(['required', 'email']);
        }
    }),
```

Use `dependsOnCreating` / `dependsOnUpdating` for mode-specific behavior. Retrieve related resource IDs: `$formData->resource(Book::uriKey(), $formData->books)`.

**Cannot be depended upon:** `File`, `Image`, `Audio`, `VaporFile`, `VaporImage`.

---

## Panels & Tabs

### Collapsible Panels

```php
use Laravel\Nova\Panel;

Panel::make('Profile', [
    Text::make('Full Name'),
    Date::make('Date of Birth'),
])->collapsible()->collapsedByDefault()->limit(1);
```

### Tab Panels (Nova 5)

```php
use Laravel\Nova\Tabs\Tab;

Tab::group('Details', [
    Tab::make('Purchases', [
        Currency::make('Price')->asMinorUnits(),
        Number::make('Tickets Available'),
    ]),
    Tab::make('Info', [Text::make('Venue')]),
]),

Tab::group('Relations', [
    HasMany::make('Orders'),
    HasManyThrough::make('Tickets'),
]),
```

Omit group titles by providing fields directly to `Tab::group()`.

---

## Repeater Fields

Store repeatable data as JSON or HasMany. Generate with `php artisan nova:repeatable LineItem`.

```php
// app/Nova/Repeaters/LineItem.php
class LineItem extends Repeatable
{
    public function fields(NovaRequest $request): array
    {
        return [
            Textarea::make('Description')->rules('required'),
            Number::make('Quantity')->rules('required', 'numeric', 'min:1'),
            Currency::make('Unit Price')->rules('required'),
        ];
    }
}

// In resource:
Repeater::make('Line Items')->repeatables([LineItem::make()])->asJson();
```

**Limitations:** No `dependsOn`, no `creationRules`/`updateRules`, no `MorphToMany` inside repeaters.

---

## Relationships

All Eloquent types supported: `HasOne`, `HasMany`, `BelongsTo`, `BelongsToMany`, `HasOneThrough`, `HasManyThrough`, `MorphOne`, `MorphMany`, `MorphTo`, `MorphToMany`.

```php
BelongsTo::make('Author', 'author', User::class)
    ->searchable()->withoutTrashed()
    ->showCreateRelationButton()->modalSize('5xl'),

HasMany::make('Posts'),  // always use plural

BelongsToMany::make('Roles')
    ->fields(fn ($request, $relatedModel) => [Text::make('Notes')])
    ->actions(fn () => [new Actions\MarkAsActive])
    ->searchable(),

Tag::make('Tags')->showCreateRelationButton()->preload(),
```

---

## Validation

```php
Text::make('Name')->rules('required', 'max:255'),

Text::make('Email')
    ->rules('required', 'email', 'max:254')
    ->creationRules('unique:users,email')
    ->updateRules('unique:users,email,{{resourceId}}'),
```

---

## Authorization

Nova leverages Laravel Policies automatically. Define a policy and Nova checks `viewAny`, `view`, `create`, `update`, `delete`, `restore`, `forceDelete`:

```php
class PostPolicy
{
    use HandlesAuthorization;

    public function viewAny(User $user): bool { return true; }
    public function create(User $user): bool { return $user->is_admin; }
    public function update(User $user, Post $post): bool { return $user->id === $post->user_id; }
    public function delete(User $user, Post $post): bool { return $user->is_admin; }
}
```

Differentiate Nova vs. application authorization:

```php
public function viewAny(User $user)
{
    return Nova::whenServing(
        fn (NovaRequest $request) => in_array('nova:view-posts', $user->permissions),
        fn ($request) => in_array('view-posts', $user->permissions),
    );
}
```

Nova 5 supports resource-specific policy classes for separate Nova authorization logic.

---

## File Fields

```php
Image::make('Photo')->disk('public')->path('photos')
    ->maxWidth(200)->rounded()->disableDownload()->prunable(),

File::make('Document')->disk('s3')
    ->storeOriginalName('document_name')->storeSize('document_size'),

File::make('Attachment')->store(fn ($request, $model) => [
    'attachment' => $request->attachment->store('/', 's3'),
    'attachment_name' => $request->attachment->getClientOriginalName(),
    'attachment_size' => $request->attachment->getSize(),
]),
```

Run `php artisan storage:link` when using the `public` disk with the local driver.

---

## Soft Deletes

Add `SoftDeletes` to your model. Nova automatically shows restore/force-delete options and a "With Trashed" filter.

---

## Common Pitfalls

- Forgetting to register policies — Nova silently allows all when no policy exists
- Singular names for `HasMany` — always use plural: `HasMany::make('Posts')`
- Not adding relationships to `$with` when used in `title()` or `subtitle()`
- Missing `asBigInt()` on `ID` for big integer primary keys
- Using `@tailwind` directives in custom components instead of `@import "tailwindcss"` (Nova 5 uses Tailwind v4)
- Defining `dependsOn` on unsupported field types (`File`, `Image`, etc.)
- Not implementing `ShouldQueue` for heavy actions, causing timeouts
- Forgetting `php artisan storage:link` for `public` disk file fields
- Using `creationRules`/`updateRules` inside Repeatable fields (not supported)
- Nova 5 requires PHP 8.1+ and Laravel 10+ — verify `composer.json` constraints
