---
layout: post
title: "Dynamic Column Positioning with Filament Tables in Laravel"
date: 2024-10-03
categories: Blog
---

"This guide outlines a solution for dynamically managing and saving table column positions in a Filament-powered Laravel application, allowing users to customize their table layouts and retain preferences across sessions."

---

### Dynamic Column Positioning with Filament Tables in Laravel

This documentation will guide you through dynamically managing and saving the positions of table columns in a Filament-powered Laravel application. This solution allows users to customize the display order of columns, save their preferences, and apply these customizations across sessions.

---

## Overview

The solution is composed of several key components:

1. **Helper Function**: A utility function to handle flattening multi-dimensional arrays like nested column names.
2. **Model**: `ColumnPosition` model, used to store and retrieve user-specific column positions.
3. **MacroServiceProvider**: Defines macros that handle setting column positions, sorting columns by position, and dynamically editing positions through a UI.
4. **Migration**: A database migration that defines the schema for the `column_positions` table, which stores user preferences.
5. **Direct `EditColumnPositionsAction` Usage**: If you already use `headerActions`, this action is available for directly editing column positions.

---

### 1. **Helper Function**

The `flattenArray()` helper function flattens a multi-dimensional array into a single-dimensional array. This helps in handling complex column name structures when saving and retrieving positions.

```php
if (! function_exists('flattenArray')) {
    function flattenArray(array $array, string $prefix = ''): array
    {
        $result = [];
        foreach ($array as $key => $value) {
            $newKey = $prefix ? "{$prefix}.{$key}" : $key;
            if (is_array($value)) {
                $result += flattenArray($value, $newKey);
            } else {
                $result[$newKey] = $value;
            }
        }
        return $result;
    }
}
```

---

### 2. **ColumnPosition Model**

The `ColumnPosition` model is used to store and retrieve the user's customized column positions. The `positions` column stores the column order as a JSON object.

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class ColumnPosition extends Model
{
    protected $fillable = ['user_id', 'table_name', 'positions'];

    protected function casts(): array
    {
        return [
            'positions' => 'array',
        ];
    }
}
```

---

### 3. **MacroServiceProvider**

In the `MacroServiceProvider`, three macros are defined:

- `position`: This macro assigns a position to each column.
- `sortColumnsByPosition`: Sorts the columns based on user preferences.
- `withColumnPositionEditor`: Adds a UI for users to edit column positions.

```php
namespace App\Providers;

use App\Models\ColumnPosition;
use Filament\Forms\Components\TextInput;
use Filament\Tables\Actions\Action;
use Filament\Tables\Columns\Column;
use Filament\Tables\Table;
use Illuminate\Support\ServiceProvider;

class MacroServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->registerColumnPositionMacro();
        $this->registerSortColumnsByPositionMacro();
        $this->registerColumnPositionEditorMacro();
    }

    /**
     * Macro to assign column position.
     */
    protected function registerColumnPositionMacro(): void
    {
        Column::macro('position', function ($position) {
            $this->position = $position;
            return $this;
        });
    }

    /**
     * Macro to sort columns based on user-saved positions.
     */
    protected function registerSortColumnsByPositionMacro(): void
    {
        Table::macro('sortColumnsByPosition', function () {
            $userId = auth()->id();
            $tableName = get_class($this->getLivewire());

            $columnPosition = ColumnPosition::where('user_id', $userId)
                ->where('table_name', $tableName)
                ->first();

            $columns = collect($this->getColumns());

            if ($columnPosition) {
                $positions = $columnPosition->positions;
                $columns->each(function ($column) use ($positions) {
                    $name = $column->getName();
                    if (isset($positions[$name])) {
                        $column->position = $positions[$name];
                    }
                });
            }

            $sortedColumns = $columns->filter(fn ($column) => isset($column->position))->sortBy('position');
            $unsortedColumns = $columns->filter(fn ($column) => ! isset($column->position));

            $this->columns($sortedColumns->merge($unsortedColumns)->toArray());

            return $this;
        });
    }

    /**
     * Macro to provide UI for editing column positions.
     */
    protected function registerColumnPositionEditorMacro(): void
    {
        Table::macro('withColumnPositionEditor', function () {
            $this->headerActions([
                Action::make('edit_column_positions')
                    ->label('Edit Column Positions')
                    ->form(function () {
                        return collect($this->getColumns())
                            ->filter(fn ($column) => isset($column->position) && $column->isVisible())
                            ->mapWithKeys(function ($column) {
                                $userId = auth()->id();
                                $tableName = get_class($this->getLivewire());

                                $columnPosition = ColumnPosition::where('user_id', $userId)
                                    ->where('table_name', $tableName)
                                    ->first();

                                $columnName = $column->getName();
                                $storedPosition = $columnPosition
                                    ? ($columnPosition->positions[$columnName] ?? $column->position)
                                    : $column->position;

                                return [
                                    $columnName => TextInput::make($columnName)
                                        ->label($column->getLabel())
                                        ->default($storedPosition)
                                        ->numeric()
                                        ->inlineLabel(),
                                ];
                            })
                            ->toArray();
                    })
                    ->action(function (array $data) {
                        $userId = auth()->id();
                        $tableName = get_class($this->getLivewire());

                        ColumnPosition::updateOrCreate(
                            ['user_id' => $userId, 'table_name' => $tableName],
                            ['positions' => flattenArray($data)]
                        );
                    }),
            ]);

            return $this;
        });
    }
}
```

---

### 4. **Migration**

The migration file for `column_positions` creates the table that stores user preferences for column positioning.

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('column_positions', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
            $table->string('table_name');
            $table->json('positions');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('column_positions');
    }
};
```

---

### 5. **Direct `EditColumnPositionsAction` Usage**

If you are already using `headerActions`, you can directly use the `EditColumnPositionsAction` class, which adds a UI to rearrange columns.

```php
namespace App\Filament\Actions\Tables;

use App\Models\ColumnPosition;
use Filament\Forms\Components\TextInput;
use Filament\Tables\Actions\Action;
use Filament\Support\Enums\MaxWidth;

class EditColumnPositionsAction extends Action
{
    public static function getDefaultName(): ?string
    {
        return 'edit_custom_positions';
    }

    protected function setUp(): void
    {
        parent::setUp();

        $this->label('Rearrange Columns')
            ->color('warning')
            ->form(function () {
                return collect($this->getTable()->getColumns())
                    ->filter(fn ($column) => isset($column->position) && $column->isVisible())
                    ->mapWithKeys(function ($column) {
                        $userId = auth()->id();
                        $tableName = get_class($this->getTable()->getLivewire());

                        $columnPosition = ColumnPosition::where('user_id', $userId)
                            ->where('table_name', $tableName)
                            ->first();

                        $columnName = $column->getName();
                        $storedPosition = $columnPosition
                            ? ($columnPosition->positions[$columnName] ?? $column->position)
                            : $column->position;

                        return [
                            $columnName => TextInput::make($columnName)
                                ->label($column->getLabel())
                                ->default($storedPosition)
                                ->numeric()
                                ->inlineLabel(),
                        ];
                    })
                    ->toArray();
            })
            ->action(function (array $data) {
                $userId = auth()->id();
                $tableName = get_class($this->getTable()->getLivewire());

                ColumnPosition::updateOrCreate(
                    ['user_id' => $userId, 'table_name' => $tableName],
                    ['positions' => flattenArray($data)]
                );
            })
            ->modalWidth(MaxWidth::ExtraLarge)
            ->extraModalFooterActions([
                Action::make('reset_column_positions')
                    ->label('Reset Positions')
                    ->color('danger')
                    ->requiresConfirmation()
                    ->slideOver(false)
                    ->action(function () {
                        $userId = auth()->id();
                        $tableName = get_class($this->getTable()->getLivewire());

                        ColumnPosition::where('user_id', $userId)
                            ->where('table_name', $tableName)
                            ->delete();
                    })->cancelParentActions('edit_custom_positions'),
            ]);
    }
}
```

---

###

6.  **Usage in Resource Table**

Here’s an example of how to apply the solution to a Filament resource table. This example includes sorting columns based on user preferences and providing a UI for editing column positions:

```php
public static function table(Table $table): Table
{
    return $table
        ->columns([
            Tables\Columns\IconColumn::make('is_active')
                ->label(__('labels.is_active'))
                ->boolean()
                ->position(1),
            Tables\Columns\TextColumn::make('name')
                ->label(__('labels.name'))
                ->searchable()
                ->position(2),
            Tables\Columns\TextColumn::make('email')
                ->label(__('labels.email'))
                ->badge()
                ->position(3),
            Tables\Columns\TextColumn::make('role')
                ->label(__('labels.role'))
                ->position(4),
            // ... other columns
        ])
        ->sortColumnsByPosition()  // Sort columns based on user preferences
        ->withColumnPositionEditor()  // Allow users to edit column positions
        ->filters([
            Tables\Filters\TrashedFilter::make(),
        ])
        ->actions([
            Tables\Actions\ViewAction::make(),
            Tables\Actions\EditAction::make(),
            Tables\Actions\DeleteAction::make(),
        ])
        ->bulkActions([
            Tables\Actions\DeleteBulkAction::make(),
            Tables\Actions\ForceDeleteBulkAction::make(),
            Tables\Actions\RestoreBulkAction::make(),
        ]);
}
```

---

### **How It Works**

- **Storing Preferences**: Users can rearrange column positions via the UI. Their preferences are saved in the `column_positions` table.
- **Applying Preferences**: The table reorders columns based on saved preferences the next time the user loads the table.
- **Editing Positions**: The `withColumnPositionEditor` macro allows users to adjust the column order in a user-friendly interface. You can also use the `EditColumnPositionsAction` directly if you already have `headerActions` defined.

This setup allows for a personalized user experience and is flexible enough to handle default behaviors for users who haven't customized their columns.
