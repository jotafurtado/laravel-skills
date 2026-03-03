# Laravel Nova 5 Skill

AI agent skill for building Laravel Nova 5 administration panels. Teaches your AI assistant how to work with Nova resources, fields, actions, filters, lenses, metrics, dashboards, authorization, and customization.

## Install

```bash
php artisan boost:add-skill your-username/laravel-nova-5-skill
```

Or with npx:

```bash
npx skills add your-username/laravel-nova-5-skill
```

## What's Covered

- Resource creation and configuration
- All field types including computed, dependent, and repeater fields
- Panels, tabs, and collapsible sections
- Relationships (all Eloquent types)
- Actions (sync, queued, batchable, standalone)
- Filters (select, boolean, date)
- Lenses with custom queries
- Metrics (Value, Trend, Partition, Progress, Table)
- Dashboards
- Authorization and policies
- Custom tools, cards, fields, and menus
- File uploads and storage
- Localization and stubs
- Nova 5 specific features (Tailwind v4, Vue 3.5, Inertia 2.x)

## Structure

```
skills/
└── laravel-nova-5/
    ├── SKILL.md                              # Main skill file
    └── references/
        ├── actions-and-filters.md            # Actions, filters, lenses
        ├── metrics-and-dashboards.md         # Metrics and dashboards
        └── customization.md                  # Tools, menus, notifications, stubs
```

## Requirements

- PHP 8.1+
- Laravel 10+
- Nova 5.x

## License

MIT
