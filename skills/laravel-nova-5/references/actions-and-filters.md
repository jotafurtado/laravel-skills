# Actions, Filters & Lenses

## Actions

Generate with `php artisan nova:action SendWelcomeEmail`.

### Defining Actions

```php
namespace App\Nova\Actions;

use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Http\Requests\NovaRequest;

class SendWelcomeEmail extends Action
{
    public $name = 'Send Welcome Email';

    public function handle(ActionFields $fields, Collection $models): mixed
    {
        foreach ($models as $model) {
            // send email logic...
        }

        return Action::message('Emails sent successfully!');
    }

    public function fields(NovaRequest $request): array
    {
        return [
            Text::make('Subject')->rules('required'),
        ];
    }
}
```

### Registering Actions

```php
public function actions(NovaRequest $request): array
{
    return [
        Actions\SendWelcomeEmail::make()
            ->canSee(fn ($request) => $request->user()->is_admin)
            ->canRun(fn ($request, $model) => $request->user()->can('email', $model))
            ->confirmText('Are you sure you want to send this email?')
            ->confirmButtonText('Send')
            ->cancelButtonText('Cancel')
            ->size('2xl'),               // sm, md, lg, xl, 2xl...7xl
    ];
}
```

### Action Responses

```php
return Action::message('Done!');
return Action::danger('Something went wrong.');
return Action::redirect('/resources/users/'.$user->id);
return Action::openInNewTab('https://example.com');
return Action::download('/path/to/file.pdf', 'report.pdf');
return Action::modal('custom-vue-component', ['data' => $value]);
```

### Queued Actions

Implement `ShouldQueue` for long-running actions:

```php
namespace App\Nova\Actions;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Collection;
use Laravel\Nova\Actions\Action;
use Laravel\Nova\Fields\ActionFields;

class GenerateReport extends Action implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function handle(ActionFields $fields, Collection $models): void
    {
        foreach ($models as $model) {
            // heavy processing...
            $this->markAsFinished($model);
        }
    }

    public function handleResult(ActionFields $fields, $results)
    {
        return Action::message('Report generation has been queued.');
    }
}
```

Use `markAsFinished($model)` and `markAsFailed($model, $exception)` to update status.

### Batchable Actions

Implement `BatchableAction` interface and use `Batchable` trait:

```php
use Illuminate\Bus\Batchable;
use Laravel\Nova\Contracts\BatchableAction;

class BulkExport extends Action implements ShouldQueue, BatchableAction
{
    use InteractsWithQueue, Queueable, Batchable;

    public function withBatch(ActionFields $fields, PendingBatch $batch): void
    {
        $batch->then(function (Batch $batch) {
            // all jobs completed...
        })->catch(function (Batch $batch, \Throwable $e) {
            // handle failure...
        });
    }
}
```

### Standalone Actions (No Resource Required)

```php
class RefreshCache extends Action
{
    public $standalone = true;

    public function handle(ActionFields $fields, Collection $models): mixed
    {
        cache()->flush();
        return Action::message('Cache cleared.');
    }
}
```

### Run Without Confirmation

```php
Actions\QuickApprove::make()->withoutConfirmation(),
```

### Built-in CSV Export

```php
use Laravel\Nova\Actions\ExportAsCsv;

ExportAsCsv::make()->withFormat(fn ($model) => [
    'ID' => $model->getKey(),
    'Name' => $model->name,
    'Email' => $model->email,
]),
```

### Action Log

Attach `Actionable` trait to your Eloquent model:

```php
use Laravel\Nova\Actions\Actionable;

class User extends Authenticatable
{
    use Actionable;
}
```

Disable logging per action: `public $withoutActionEvents = true;`.

### Action Authorization Priority

1. `canRun` method on the action (if defined)
2. Model policy's `runAction` / `runDestructiveAction` methods (if defined)
3. Model policy's `update` / `delete` methods

---

## Filters

Generate with `php artisan nova:filter UserType`.

### Defining Filters

```php
namespace App\Nova\Filters;

use Illuminate\Database\Eloquent\Builder;
use Laravel\Nova\Filters\Filter;
use Laravel\Nova\Http\Requests\NovaRequest;

class UserType extends Filter
{
    public function apply(NovaRequest $request, Builder $query, mixed $value): Builder
    {
        return $query->where('type', $value);
    }

    public function options(NovaRequest $request): array
    {
        return [
            'Admin' => 'admin',
            'Editor' => 'editor',
            'Viewer' => 'viewer',
        ];
    }
}
```

### Registering Filters

```php
public function filters(NovaRequest $request): array
{
    return [
        new Filters\UserType,
    ];
}
```

Nova 5 supports **searchable select filters** out of the box.

### Boolean Filter

```php
use Laravel\Nova\Filters\BooleanFilter;

class ActiveUsers extends BooleanFilter
{
    public function apply(NovaRequest $request, Builder $query, mixed $value): Builder
    {
        if ($value['active'] ?? false) {
            return $query->where('active', true);
        }

        return $query;
    }

    public function options(NovaRequest $request): array
    {
        return ['Active' => 'active'];
    }
}
```

### Date Filter

```php
use Laravel\Nova\Filters\DateFilter;

class CreatedAfter extends DateFilter
{
    public function apply(NovaRequest $request, Builder $query, mixed $value): Builder
    {
        return $query->whereDate('created_at', '>=', $value);
    }
}
```

---

## Lenses

Lenses provide specialized resource views with custom Eloquent queries:

```bash
php artisan nova:lens MostValuableUsers
```

```php
namespace App\Nova\Lenses;

use Laravel\Nova\Fields\ID;
use Laravel\Nova\Fields\Number;
use Laravel\Nova\Fields\Text;
use Laravel\Nova\Http\Requests\LensRequest;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Lenses\Lens;

class MostValuableUsers extends Lens
{
    public static function query(LensRequest $request, $query)
    {
        return $request->withOrdering($request->withFilters(
            $query->select('users.id', 'users.name')
                  ->selectRaw('sum(licenses.price) as revenue')
                  ->join('licenses', 'users.id', '=', 'licenses.user_id')
                  ->groupBy('users.id', 'users.name')
                  ->orderByDesc('revenue')
        ));
    }

    public function fields(NovaRequest $request): array
    {
        return [
            ID::make('ID', 'id'),
            Text::make('Name'),
            Number::make('Revenue')->sortable(),
        ];
    }

    public function filters(NovaRequest $request): array
    {
        return [];
    }

    public function actions(NovaRequest $request): array
    {
        return [];
    }

    public function cards(NovaRequest $request): array
    {
        return [new \App\Nova\Metrics\NewUsers];
    }
}
```

### Registering Lenses

```php
public function lenses(NovaRequest $request): array
{
    return [
        Lenses\MostValuableUsers::make()
            ->canSee(fn ($request) => $request->user()->is_admin),
    ];
}
```

### Lens Polling

```php
public static $polling = true;
public static $pollingInterval = 10;
public static $showPollingToggle = true;
```

### Per-Page Options on Lens

```php
public static $perPageOptions = [25, 50, 100];
```
