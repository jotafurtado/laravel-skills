# Customization: Tools, Menus, Notifications & Stubs

## Custom Tools

Build full-page tools with their own Vue component and routes:

```bash
php artisan nova:tool acme/analytics-dashboard
```

Register in `NovaServiceProvider`:

```php
public function tools(): array
{
    return [
        new \Acme\AnalyticsDashboard\AnalyticsDashboard,
    ];
}
```

Each tool includes its own service provider and a single-file Vue component. Tool authorization:

```php
(new AnalyticsDashboard)->canSee(fn ($request) => $request->user()->is_admin),
```

---

## Resource Tools

Resource tools appear on a resource's detail page (not the sidebar):

```bash
php artisan nova:resource-tool acme/stripe-inspector
```

Register in a resource's `fields()` method:

```php
use Acme\StripeInspector\StripeInspector;

public function fields(NovaRequest $request): array
{
    return [
        ID::make()->sortable(),

        StripeInspector::make()
            ->issuesRefunds()  // dynamic option
            ->canSee(fn ($request) => $request->user()->is_admin),
    ];
}
```

Dynamic options: call any method on the tool instance to set key-value options accessible in the Vue component via `panel.fields`.

---

## Custom Cards

```bash
php artisan nova:card acme/weather-widget
```

Register cards in a resource's `cards()` method or in a dashboard.

---

## Custom Fields

```bash
php artisan nova:field acme/color-picker
```

Creates a custom field with its own Vue component for index, detail, and form views.

---

## Custom Filters

```bash
php artisan nova:filter acme/date-range-filter
```

---

## Menus & Navigation

Customize the Nova sidebar in `NovaServiceProvider`:

```php
use Laravel\Nova\Menu\MenuItem;
use Laravel\Nova\Menu\MenuSection;
use Laravel\Nova\Nova;

Nova::mainMenu(fn ($request) => [
    MenuSection::dashboard(Main::class)->icon('chart-bar'),

    MenuSection::make('Content', [
        MenuItem::resource(Post::class),
        MenuItem::resource(Category::class),
    ])->icon('document-text')->collapsable(),

    MenuSection::make('Users', [
        MenuItem::resource(User::class),
        MenuItem::resource(Role::class),
    ])->icon('users')->collapsable(),

    MenuSection::make('External', [
        MenuItem::externalLink('Documentation', 'https://nova.laravel.com/docs'),
    ])->icon('book-open'),
]);
```

### User Menu

```php
Nova::userMenu(fn ($request, $menu) => $menu
    ->prepend(MenuItem::link('My Profile', '/resources/users/'.$request->user()->getKey()))
    ->append(MenuItem::externalLink('Help', 'https://help.example.com'))
);
```

---

## Notifications

Nova includes a built-in notification center. Send notifications to users:

```php
use Laravel\Nova\Notifications\NovaNotification;
use Laravel\Nova\URL;

$user->notify(
    NovaNotification::make()
        ->message('Your report is ready!')
        ->action('Download', URL::remote('https://example.com/report.pdf'))
        ->icon('download')
        ->type('info')  // info, success, warning, error
);
```

---

## Authentication

Nova 5 integrates with Laravel Fortify. Configure in `config/nova.php`:

```php
'guard' => 'web',

'passwords' => 'users',
```

### Gate Authorization

In `NovaServiceProvider`, define who can access Nova:

```php
protected function gate(): void
{
    Gate::define('viewNova', function (User $user) {
        return in_array($user->email, [
            'admin@example.com',
        ]);
    });
}
```

---

## Impersonation

Enable user impersonation:

```php
// In NovaServiceProvider
Nova::impersonation(true);
```

---

## Localization

Publish and customize Nova translations:

```bash
php artisan nova:translate pt-BR
```

Or manually create `lang/vendor/nova/pt-BR.json`:

```json
{
    "Create :resource": "Criar :resource",
    "Update :resource": "Atualizar :resource",
    "Delete": "Excluir"
}
```

---

## Stubs

Publish and customize Nova stubs:

```bash
php artisan nova:stubs
```

Publishes stubs to `stubs/nova/` for full customization of generated files.

---

## Assets (CSS / JavaScript)

Register custom styles and scripts in `NovaServiceProvider`:

```php
Nova::style('custom-theme', public_path('css/nova-custom.css'));
Nova::script('custom-scripts', public_path('js/nova-custom.js'));
```

---

## Global Search

Configure searchable columns per resource:

```php
public static $search = ['id', 'name', 'email'];

// Full-text search via Scout:
public static $searchUsing = 'scout';
```

Customize the global search result subtitle:

```php
public function subtitle(): ?string
{
    return "Author: {$this->user->name}";
}
```

---

## Resource Replication

Override to customize clone behavior:

```php
public function replicate()
{
    return tap(parent::replicate(), function ($resource) {
        $resource->model()->name = 'Copy of '.$resource->model()->name;
    });
}
```

**Note:** Markdown/Trix fields with `withFiles` may not be replicated.

---

## Configuration

Publish the Nova config:

```bash
php artisan vendor:publish --tag=nova-config
```

Key options in `config/nova.php`:

```php
return [
    'path' => '/nova',             // URL prefix
    'guard' => 'web',              // auth guard
    'passwords' => 'users',        // password broker
    'currency' => 'BRL',           // default currency
    'brand' => [
        'name' => 'My App',
        'logo' => '/img/logo.svg',
    ],
    'pagination' => 'simple',      // simple or links
];
```
