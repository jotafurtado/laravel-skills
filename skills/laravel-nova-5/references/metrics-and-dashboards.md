# Metrics & Dashboards

## Metrics Overview

Nova offers five metric types: Value, Trend, Partition, Progress, and Table.

---

## Value Metric

Displays a single value with optional comparison to a prior period.

```bash
php artisan nova:value NewUsers
```

```php
namespace App\Nova\Metrics;

use App\Models\User;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Value;
use Laravel\Nova\Metrics\ValueResult;

class NewUsers extends Value
{
    public function calculate(NovaRequest $request): ValueResult
    {
        return $this->count($request, User::class);
    }

    public function ranges(): array
    {
        return [
            30 => '30 Days',
            60 => '60 Days',
            365 => '365 Days',
        ];
    }

    public function cacheFor(): ?\DateTimeInterface
    {
        return now()->addMinutes(5);
    }

    public function name(): string
    {
        return 'Users Created';
    }
}
```

Value query helpers: `count`, `average`, `sum`, `max`, `min`. Each accepts `($request, $model, $column)`.

---

## Trend Metric

Displays data over time as a line/bar chart.

```bash
php artisan nova:trend UsersPerDay
```

```php
namespace App\Nova\Metrics;

use App\Models\User;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Trend;
use Laravel\Nova\Metrics\TrendResult;

class UsersPerDay extends Trend
{
    public function calculate(NovaRequest $request): TrendResult
    {
        return $this->countByDays($request, User::class);
    }

    public function ranges(): array
    {
        return [
            7 => '7 Days',
            14 => '14 Days',
            30 => '30 Days',
        ];
    }
}
```

Trend query helpers: `countByDays`, `countByWeeks`, `countByMonths`, `countByHours`, `countByMinutes`, `sumByDays`, `sumByWeeks`, `sumByMonths`, `averageByDays`, etc.

---

## Partition Metric

Displays a doughnut/pie chart for categorical data.

```bash
php artisan nova:partition UsersPerPlan
```

```php
namespace App\Nova\Metrics;

use App\Models\User;
use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Partition;
use Laravel\Nova\Metrics\PartitionResult;

class UsersPerPlan extends Partition
{
    public function calculate(NovaRequest $request): PartitionResult
    {
        return $this->count($request, User::class, 'plan')
            ->label(fn ($value) => match ($value) {
                'basic' => 'Basic',
                'pro' => 'Professional',
                default => ucfirst($value),
            })
            ->colors([
                'basic' => '#4ade80',
                'pro' => '#3b82f6',
            ]);
    }
}
```

---

## Progress Metric

Displays a progress bar toward a goal.

```bash
php artisan nova:progress OnboardingCompletion
```

```php
namespace App\Nova\Metrics;

use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\Progress;
use Laravel\Nova\Metrics\ProgressResult;

class OnboardingCompletion extends Progress
{
    public function calculate(NovaRequest $request): ProgressResult
    {
        return $this->result(80, 100); // current, target
    }
}
```

---

## Table Metric

Displays a list of rows with titles, subtitles, and optional actions/icons.

```bash
php artisan nova:table NewReleases
```

```php
namespace App\Nova\Metrics;

use Laravel\Nova\Http\Requests\NovaRequest;
use Laravel\Nova\Metrics\MetricTableRow;
use Laravel\Nova\Metrics\Table;

class NewReleases extends Table
{
    public function calculate(NovaRequest $request): array
    {
        return [
            MetricTableRow::make()
                ->title('Nova 5.0')
                ->subtitle('Released January 2025')
                ->icon('star'),

            MetricTableRow::make()
                ->title('Laravel 11.x')
                ->subtitle('Latest framework release'),
        ];
    }
}
```

---

## Registering Metrics

### On Resource Index

```php
public function cards(NovaRequest $request): array
{
    return [
        new Metrics\NewUsers,
        new Metrics\UsersPerDay,
    ];
}
```

### On Resource Detail

```php
Metrics\UserRevenue::make()->onlyOnDetail(),
```

### Metric Options

```php
Metrics\TotalUsers::make()
    ->refreshWhenActionsRun()      // auto-refresh after actions
    ->refreshWhenFiltersChange()   // auto-refresh on filter change
    ->defaultRange(30)             // initial range selection
    ->width('1/2'),                // 1/3, 1/2, 2/3, full
```

### Metric Caching

```php
public function cacheFor(): ?\DateTimeInterface
{
    return now()->addMinutes(5);
    // or return now()->addHours(1);
    // or return null; // no caching
}
```

### Authorization

```php
Metrics\UsersPerDay::make()
    ->canSee(fn ($request) => $request->user()->is_admin),

// Or using policy shorthand:
Metrics\UsersPerDay::make()
    ->canSeeWhen('viewUsersPerDay', User::class),
```

---

## Sparkline Field

Display inline charts on index/detail using Trend metrics or raw data:

```php
use Laravel\Nova\Fields\Sparkline;

Sparkline::make('Post Views')->data([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]),

Sparkline::make('Post Views')->data(fn () => json_decode($this->views_data)),

Sparkline::make('Post Views')->data(new PostViewsOverTime($this->id)),
```

---

## Dashboards

Custom dashboards display metric cards outside of any resource context.

```bash
php artisan nova:dashboard Main
```

```php
namespace App\Nova\Dashboards;

use App\Nova\Metrics\NewUsers;
use App\Nova\Metrics\UsersPerDay;
use App\Nova\Metrics\UsersPerPlan;
use Laravel\Nova\Dashboards\Main as Dashboard;
use Laravel\Nova\Http\Requests\NovaRequest;

class Main extends Dashboard
{
    public function cards(): array
    {
        return [
            (new NewUsers)->width('1/3'),
            (new UsersPerDay)->width('2/3'),
            (new UsersPerPlan)->width('1/2'),
        ];
    }

    public static function label(): string
    {
        return 'Overview';
    }
}
```

Register dashboards in `NovaServiceProvider`:

```php
protected function dashboards(): array
{
    return [
        new \App\Nova\Dashboards\Main,
    ];
}
```

Dashboard authorization:

```php
public function authorize(Request $request): bool
{
    return $request->user()->is_admin;
}
```
