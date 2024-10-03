---
layout: post
title: "Dynamic Column Positioning with Filament Tables in Laravel"
date: 2024-10-01 14:00:00 +0000
categories: laravel filament tables dynamic-columns
---

Here’s a detailed documentation explaining the working and usage of the provided solution, covering each part of the code:

---

### Dynamic Column Positioning with Filament Tables

This documentation explains how to dynamically manage and save the positions of table columns in a Filament-powered Laravel application. The goal is to allow users to customize the display order of columns, save their preferences, and apply these customizations across sessions.

---

## Overview

The solution is comprised of several key components:

1. **Helper Function**: `flattenArray()` to handle flattening multi-dimensional arrays like nested column names.
2. **Model**: `ColumnPosition` model to store and retrieve the user-specific column positions.
3. **MacroServiceProvider**: Defines macros that allow for setting column positions, sorting columns by position, and dynamically editing positions through a UI.
4. **Migration**: Defines the `column_positions` table schema for storing user preferences.

---

### 1. **Helper Function**

The `flattenArray()` helper function flattens a multi-dimensional array into a single-dimensional array. For example, it converts nested keys like `insuranceProducts.name` into `insuranceProducts.name`.

#### Code:

```php
if (! function_exists('flattenArray')) {
    function flattenArray(array $array, string $prefix = ''): array
    {
        $result = [];
        foreach ($array as $key => $value) {
            // Create a new key based on the prefix
            $newKey = $prefix ? "{$prefix}.{$key}" : $key;

            // Recursively flatten arrays
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

The `ColumnPosition` model is used to store and retrieve the column order preferences for each user and table. Each entry includes the user ID, the table name (to distinguish between different resources or tables), and a JSON object storing the positions of columns.

#### Code:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class ColumnPosition extends Model
{
    protected $fillable = [
        'user_id',
        'table_name',
        'positions',
    ];

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

The `MacroServiceProvider` defines macros for Filament's `Column` and `Table` classes. These macros handle column positioning and saving/restoring positions in a user-specific manner.

#### Key Features:

- **`position` Macro**: Assigns a position to a column.
- **`sortColumnsByPosition` Macro**: Sorts columns by their stored position from the `ColumnPosition` model or based on default positions.
- **`withColumnPositionEditor` Macro**: Adds a UI action that allows users to edit column positions dynamically.

#### Code:

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
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        // Macro to assign position to columns dynamically
        Column::macro('position', function ($position) {
            $this->position = $position;
            return $this;
        });

        // Macro to sort columns by position from the database
        Table::macro('sortColumnsByPosition', function () {
            $userId = auth()->id();
            $tableName = get_class($this->getLivewire());

            // Fetch user-specific column positions from the database
            $columnPosition = ColumnPosition::where('user_id', $userId)
                ->where('table_name', $tableName)
                ->first();

            // Get current columns
            $columns = collect($this->getColumns());

            // Apply saved positions if they exist
            if ($columnPosition) {
                $positions = $columnPosition->positions;

                // Assign stored positions to columns
                $columns->each(function ($column) use ($positions) {
                    $name = $column->getName();
                    if (isset($positions[$name])) {
                        $column->position = $positions[$name];
                    }
                });
            }

            // Sort columns with positions
            $sortedColumns = $columns->filter(fn ($column) => isset($column->position))->sortBy('position');

            // Add unsorted columns at the end
            $unsortedColumns = $columns->filter(fn ($column) => ! isset($column->position));
            $finalColumns = $sortedColumns->merge($unsortedColumns);

            // Set the final ordered columns back to the table
            $this->columns($finalColumns->toArray());

            return $this;
        });

        // Macro to allow editing column positions through the UI
        Table::macro('withColumnPositionEditor', function () {
            $this->headerActions([
                Action::make('edit_column_positions')
                    ->label('Edit Column Positions')
                    ->form(function () {
                        return collect($this->getColumns())
                            ->filter(fn ($column) => isset($column->position)) // Show only columns with a position
                            ->mapWithKeys(function ($column) {
                                $userId = auth()->id();
                                $tableName = get_class($this->getLivewire());

                                // Fetch stored column positions
                                $columnPosition = ColumnPosition::where('user_id', $userId)
                                    ->where('table_name', $tableName)
                                    ->first();

                                $columnName = $column->getName();
                                $storedPosition = $columnPosition ? ($columnPosition->positions[$columnName] ?? $column->position) : $column->position;

                                return [
                                    $columnName => TextInput::make($columnName)
                                        ->label($column->getLabel())
                                        ->default($storedPosition)
                                        ->inlineLabel(),
                                ];
                            })
                            ->toArray();
                    })
                    ->action(function (array $data) {
                        $userId = auth()->id();
                        $tableName = get_class($this->getLivewire());

                        // Flatten nested column names
                        $flattenedData = flattenArray($data);

                        // Save or update positions in the ColumnPosition model
                        ColumnPosition::updateOrCreate(
                            [
                                'user_id' => $userId,
                                'table_name' => $tableName,
                            ],
                            ['positions' => $flattenedData]
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

This migration creates the `column_positions` table, which stores the column order preferences of each user.

#### Code:

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

### 5. **Usage in Resource Table**

In your resource class, use the `sortColumnsByPosition()` and `withColumnPositionEditor()` methods after defining the columns to enable dynamic column sorting and position editing.

#### Example:

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

### **How it Works**

- **Storing Preferences**: When a user customizes the column positions through the UI, their preferences are saved in the `column_positions` table with the `user_id`, `table_name`, and the column positions.
- **Applying Preferences**: On subsequent loads, the saved column positions are applied to the table, and the columns are reordered accordingly.
- **Editing Positions**: The `withColumnPositionEditor` macro provides a simple UI for users to adjust the order of columns and save their preferences.
